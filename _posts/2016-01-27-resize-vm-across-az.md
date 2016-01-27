---

layout: post
title: 实例resize跨可用域问题

---
问题：
在实际使用openstack的过程中发现实例resize操作的时候存在跨可用域的问题。

问题分析：
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
