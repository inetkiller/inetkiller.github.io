---

layout: post
title: openstack horizon不支持根据实例IP地址进行搜索的问题

---

**问题：**

我用的是openstack juno版本，以admin用户登陆dashboard，点击管理员panel group,再点击实例panel，里面有一些搜索filter可选，其中根据ipv4,ipv6,project这三个filter进行搜索只会从当前显示的页面中搜索，实际使用过程中非常难用。

![](http://ww3.sinaimg.cn/mw690/5d326ffdgw1f0e5i1wesdj21kw0mbdmx.jpg)

**问题分析：**

在没有看horizon的代码以前，我一直以为horizon是一次性从nova-api中取到所有的实例信息，然后进行搜索。网上有一些资料是这样说的，horizon支持两种类型的搜索，一个是server类型（在horizon里搜索），一个是前端使用js搜索。可能以前的老版本是这样做的，不过至少在juno版本的代码中搜索的类型已经变化了，现在也有两种，一种是api类型，一种是server类型。

api类型：horizon在调用nova-api的时候直接传递相关的filter，由nova来搜索并返回符合相关filter的实例给horizon，horizon也并不是一次性向nova-api请求所有的实例信息，而是传递给nova-api一个用户翻页信息，nova-api每次只返回给horizon一页的信息。

server类型：horizon不向nova-api传递filter信息，nova-api直接返回给horizon未经筛选的一页信息，horizon收到该部分数据后再进行filter，表示出来就像是基于前端js写的搜索，只能搜索当前页的，作用并不大。

因此可以猜测，ipv4,ipv6,project这三个filter是server类型的吗？

查看代码后发现project是server类型的，而ipv4,ipv6都是api类型，那么project的表现就是符合预期的，ipv4,ipv6 filter明显还是有问题。

horizon请求实例信息使用的nova api接口是“/v2.1/{tenant_id}/servers/detail”，可接受的参数如下图：

![](http://ww1.sinaimg.cn/mw690/5d326ffdgw1f0e68z3bvaj20o70l642p.jpg)

里面并没有看到支持根据IP进行搜索的相关参数。难道说nova-api根本不支持根据IP进行搜索吗？有一段时间我的结论就是这样的，直到有一天我继续看了“/v2.1/{tenant_id}/servers/detail”这个接口的代码。发现虽然在文档中没有给出这个参数，但实际在代码中是可以处理这个参数的，最终我定位到在nova-api中处理ip filter的方法是下面这个：

{% highlight python %}
def get_all(self, context, search_opts=None, limit=None, marker=None,
                want_objects=False, expected_attrs=None, sort_keys=None,
                sort_dirs=None):
        """Get all instances filtered by one of the given parameters.

        If there is no filter and the context is an admin, it will retrieve
        all instances in the system.

        Deleted instances will be returned by default, unless there is a
        search option that says otherwise.

        The results will be sorted based on the list of sort keys in the
        'sort_keys' parameter (first value is primary sort key, second value is
        secondary sort ket, etc.). For each sort key, the associated sort
        direction is based on the list of sort directions in the 'sort_dirs'
        parameter.
        """

        # TODO(bcwaldon): determine the best argument for target here
        target = {
            'project_id': context.project_id,
            'user_id': context.user_id,
        }

        if not self.skip_policy_check:
            check_policy(context, "get_all", target)

        if search_opts is None:
            search_opts = {}

        LOG.debug("Searching by: %s" % str(search_opts))

        # Fixups for the DB call
        filters = {}

        def _remap_flavor_filter(flavor_id):
            flavor = objects.Flavor.get_by_flavor_id(context, flavor_id)
            filters['instance_type_id'] = flavor.id

        def _remap_fixed_ip_filter(fixed_ip):
            # Turn fixed_ip into a regexp match. Since '.' matches
            # any character, we need to use regexp escaping for it.
            filters['ip'] = '^%s$' % fixed_ip.replace('.', '\\.')

        def _remap_metadata_filter(metadata):
            filters['metadata'] = jsonutils.loads(metadata)

        def _remap_system_metadata_filter(metadata):
            filters['system_metadata'] = jsonutils.loads(metadata)

        # search_option to filter_name mapping.
        filter_mapping = {
                'image': 'image_ref',
                'name': 'display_name',
                'tenant_id': 'project_id',
                'flavor': _remap_flavor_filter,
                'fixed_ip': _remap_fixed_ip_filter,
                'metadata': _remap_metadata_filter,
                'system_metadata': _remap_system_metadata_filter}

        # copy from search_opts, doing various remappings as necessary
        for opt, value in six.iteritems(search_opts):
            # Do remappings.
            # Values not in the filter_mapping table are copied as-is.
            # If remapping is None, option is not copied
            # If the remapping is a string, it is the filter_name to use
            try:
                remap_object = filter_mapping[opt]
            except KeyError:
                filters[opt] = value
            else:
                # Remaps are strings to translate to, or functions to call
                # to do the translating as defined by the table above.
                if isinstance(remap_object, six.string_types):
                    filters[remap_object] = value
                else:
                    try:
                        remap_object(value)

                    # We already know we can't match the filter, so
                    # return an empty list
                    except ValueError:
                        return []

        # IP address filtering cannot be applied at the DB layer, remove any DB
        # limit so that it can be applied after the IP filter.
        filter_ip = 'ip6' in filters or 'ip' in filters
        orig_limit = limit
        ##if filter_ip and limit:
        ##    LOG.debug('Removing limit for DB query due to IP filter')
        ##    limit = None

        inst_models = self._get_instances_by_filters(context, filters,
                limit=limit, marker=marker, expected_attrs=expected_attrs,
                sort_keys=sort_keys, sort_dirs=sort_dirs)

        if filter_ip:
            inst_models = self._ip_filter(inst_models, filters, orig_limit)

        if want_objects:
            return inst_models

        # Convert the models to dictionaries
        instances = []
        for inst_model in inst_models:
            instances.append(obj_base.obj_to_primitive(inst_model))

        return instances
{% endhighlight %}

其中代码前面有##的三行代码在最开始排查这个问题的时候是没有的。最后的解决方法就是增加了这三行代码，当然这是后话。这三行代码的作用就是检测到horizon请求nova-api的参数中如果有ip或者ip6则将limit置为None。为什么将limit置为None就可以解决这个问题呢？

查看nova数据库中的instances表发现里面并没有实例的IP地址等网络信息（nova-api里支持的那些参数在instances表中是有的），而是通过在fixed_ips这个表里关联实例的UUID来实现实例与IP的映射。于是我猜测上面方法中的self._get_instances_by_filters方法并不支持直接指定ip，ip6 filter进行搜索，需要在外层做一些处理，而在外层做处理的前提是让self._get_instances_by_filters一次返回所有的实例信息（如果一次也只返回一页的信息，那么就跟在horizon端实现搜索无区别了），然后再对这些信息进行处理，这个外层处理的方法就是self._ip_filter(inst_models, filters, orig_limit)代码如下：

{% highlight python %}
@staticmethod
    def _ip_filter(inst_models, filters, limit):
        ipv4_f = re.compile(str(filters.get('ip')))
        ipv6_f = re.compile(str(filters.get('ip6')))

        def _match_instance(instance):
            nw_info = compute_utils.get_nw_info_for_instance(instance)
            for vif in nw_info:
                for fixed_ip in vif.fixed_ips():
                    address = fixed_ip.get('address')
                    if not address:
                        continue
                    version = fixed_ip.get('version')
                    if ((version == 4 and ipv4_f.match(address)) or
                        (version == 6 and ipv6_f.match(address))):
                        return True
            return False

        result_objs = []
        for instance in inst_models:
            if _match_instance(instance):
                result_objs.append(instance)
                if limit and len(result_objs) == limit:
                    break
        return objects.InstanceList(objects=result_objs)
{% endhighlight %}

可以看到，在这个方法中对self._get_instances_by_filters返回的所有实例的IP信息进行了匹配，同时也对返回给horizon的实例数量做了处理。

**解决方法：**

在问题分析中已经给出，增加三行代码而以。

**总结：**

在部署openstack juno版的过程中我用的是openstack的官方文档中的yum源，这个源里面的一些openstack组件的代码并不是最新版本（与github上的juno stable不同步），所以导致了这个问题的出现。看来以后部署环境还是从github上拉稳定版本源码安装比较靠谱。
