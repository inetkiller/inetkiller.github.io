---

layout: post
title: 实例resize跨可用域问题

---
**问题：**

在实际使用openstack的过程中发现实例resize操作的时候存在跨可用域的问题。

**问题分析：**

resize的时候找符合要求的计算结点这个工作是由nova-schduler来完成的。scheduler会对所有的计算结点做一遍filter，然后对通过filter的主机做weight。问题就出在filter这里，默认的filter是包括AvailabilityZoneFilter的，也就是说，如果计算结点不满足实例所属的az的话，那么它不会通过filter，实例当然也就不会resize到这台计算结点上，表现出来的功能就是az之间的schduler是隔离的。
AvailabilityZoneFilter的源码：
{% highlight python %}

class AvailabilityZoneFilter(filters.BaseHostFilter):
    """Filters Hosts by availability zone.

    Works with aggregate metadata availability zones, using the key
    'availability_zone'
    Note: in theory a compute node can be part of multiple availability_zones
    """

    # Availability zones do not change within a request
    run_filter_once_per_request = True

    def host_passes(self, host_state, filter_properties):
        spec = filter_properties.get('request_spec', {})
        props = spec.get('instance_properties', {})
        availability_zone = props.get('availability_zone')

        if not availability_zone:
            return True

        context = filter_properties['context']
        metadata = db.aggregate_metadata_get_by_host(
                context, host_state.host, key='availability_zone')

        if 'availability_zone' in metadata:
            hosts_passes = availability_zone in metadata['availability_zone']
            host_az = metadata['availability_zone']
        else:
            hosts_passes = availability_zone == CONF.default_availability_zone
            host_az = CONF.default_availability_zone

        if not hosts_passes:
            LOG.debug("Availability Zone '%(az)s' requested. "
                      "%(host_state)s has AZs: %(host_az)s",
                      {'host_state': host_state,
                       'az': availability_zone,
                       'host_az': host_az})

        return hosts_passes

{% endhighlight %}

里面这个host_passes方法就是一个过滤器，每一个计算结点都要经过这个过滤器的筛选，如果host_passes返回真，则表示该计算结点通过了这个过滤器，可以进入下一个过滤器或者进行weight了，若返回假，表示该计算结点未通过过滤器，实例不会被调度到该结点上。

再详细分析一下上述代码。
下面的代码是取到实例所属的az(可用域),如果实例的az为空，即不属于任何az，那么直接返回真（所有的计算结点都可以通过这个filter）:
{% highlight python %}
spec = filter_properties.get('request_spec', {})
props = spec.get('instance_properties', {})
availability_zone = props.get('availability_zone')

if not availability_zone:
            return True
{% endhighlight %}

下面这部分是取到计算结点的az:
{% highlight python %}
context = filter_properties['context']
metadata = db.aggregate_metadata_get_by_host(
        context, host_state.host, key='availability_zone')
{% endhighlight %}

下面这部分是比较实例的az和当前被筛选的计算结点的az是否一致，一致返回真，否则返回假（注意，有些计算结点属于默认的nova可用域）。
{% highlight python %}
        if 'availability_zone' in metadata:
            hosts_passes = availability_zone in metadata['availability_zone']
            host_az = metadata['availability_zone']
        else:
            hosts_passes = availability_zone == CONF.default_availability_zone
            host_az = CONF.default_availability_zone
{% endhighlight %}

经过分析上面的代码，以及对数据库的检查，出现问题的原因很明显是因为实例的az信息为空，导致了所有计算结点都通过了这个filter，所以resize的时候存在跨可用域的现象。

那么为什么实例的az信息会为空呢？

通过查看nova数据库中的instances表发现，有些实例az为空，有些不为空。仔细回忆想起来，这些为空的实例当时都是通过nova boot以命令行的方式来创建的，在参数中并没有指定可用域，会不会是不明确指定可用域，实例的可用域就是空的呢？后面经过实验验证了这一猜测。

为什么实例可用域为空这件事情居然没有被我发现呢？

因为通过命令行创建实例后，我在dashboard里确认了一下信息，看到实例的可用域显示正常。

难道dashboard显示实例的可用域信息不是直接从instances这个表中取到的吗？

确实不是直接从instances表中取到的。dashboard获得实例信息是请求的nova-api，经过查看dashboard与nova-api相关部分的代码，发现nova-api有扩展api这一说(具体实现还是比较复杂的，这里不深入讨论),实际上nova-api返回给dashboard实例所属的az信息是通过下面这一段代码来做的：
{% highlight python %}
def get_instance_availability_zone(context, instance):
    """Return availability zone of specified instance."""
    host = str(instance.get('host'))
    if not host:
        return None

    cache_key = _make_cache_key(host)
    cache = _get_cache()
    az = cache.get(cache_key)
    if not az:
        elevated = context.elevated()
        az = get_host_availability_zone(elevated, host)
        cache.set(cache_key, az, AZ_CACHE_SECONDS)
    return az
{% endhighlight %}
这段代码的逻辑是先得到实例在哪台计算结点上，然后再取得这个计算结点属于哪个可用域，然后做为实例的可用域信息返回给dashboard。

原来是这个扩展api返回的数据欺骗了我的双眼。。。


**解决方法：**

以后用命令行创建实例的时候一定要明确指定可用域信息。对于已经存在的实例，可以去instances表中直接改实例的可用域信息（该方法未经过测试）。
