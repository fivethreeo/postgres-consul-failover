
git clone https://github.com/fivethreeo/postgres-consul-failover.git
cd postgres-consul-failover

Use install.sh
==============

::

  ./install.sh nodename server|client master|slave

or

Install packages
================

::

    apt-get install keepalived haproxy


Install consul binary
=====================

::

    wget https://releases.hashicorp.com/consul/0.6.3/consul_0.6.3_linux_amd64.zip
    unzip consul_0.6.3_linux_amd64.zip
    cp consul /bin/
    chmod gu=rx /bin/consul


Install consul-template binary
=============================

::

    wget https://releases.hashicorp.com/consul-template/0.12.2/consul-template_0.12.2_linux_amd64.zip
    unzip consul-template_0.12.2_linux_amd64.zip
    cp consul-template /bin


Configure consul and consul-template binary
===========================================

::

    clientserver=server # server or client
    masterslave=master # master or slave
    nodename=agent-one # unique nodenamre

    priority=101
    if [ "$masterslave" == "master" ]; then
    	priority=100
    fi
    if [ "$clientserver" == "server" ]; then
    	serverarg="-server "
    fi

    ip=`ifconfig eth0 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}'`
    echo $ip
    echo $serverarg

    mkdir -p /etc/consul
    mkdir -p /tmp/consul

    echo "DAEMON_ARGS=\"agent ${serverarg}-node=${nodename} -bind=${ip} -config-dir=/etc/consul -data-dir=/tmp/consul\"" > /etc/default/consul 

    cp consul-init /etc/init.d/consul
    chmod gu=rx /etc/init.d/consul
    /etc/init.d/consul start

    mkdir -p /etc/consul-template

    echo "global
            log 127.0.0.1   local0
            log 127.0.0.1   local1 notice
            maxconn 4096
            user haproxy
            group haproxy

    defaults
            log     global
            mode    http
            option  httplog
            option  dontlognull
            retries 3
            option redispatch
            maxconn 2000
            contimeout      5000
            clitimeout      50000
            srvtimeout      50000" > /etc/haproxy/haproxy.ctmpl


    echo "consul = \"127.0.0.1:8500\"

    template {
      source = \"/etc/haproxy/haproxy.ctmpl\"
      destination = \"/etc/haproxy/haproxy.cfg\"
      command = \"service haproxy restart\"
    }" > /etc/consul-template/consul-template.conf

    cp consul-template-init /etc/init.d/consul-template

    sed -i -e "s/\(NAME=consul\)$/\1-template/" /etc/init.d/consul-template
    sed -i -e "s/\(\/etc\/consul\)\"/\1-template\"/" /etc/init.d/consul-template

    echo "DAEMON_ARGS=\"\-config /etc/consul-template/consul-template.conf\"" /etc/default/consul-template

    chmod gu=rx /etc/init.d/consul-template
    /etc/init.d/consul-template start

    sed -i -e "s/\(ENABLED\=\)0/\11/" /etc/default/haproxy

Configure keepalived
====================

::

    if ! grep -q "net.ipv4.ip_nonlocal_bind=1" /etc/sysctl.conf ; then
        echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
    fi

    sysctl -p

    echo "vrrp_script chk_haproxy {           # Requires keepalived-1.1.13
            script "killall -0 haproxy"     # cheaper than pidof
            interval 2                      # check every 2 seconds
            weight 2                        # add 2 points of prio if OK
    }

    vrrp_instance VI_1 {
            interface eth0
            state MASTER
            virtual_router_id 51
            priority ${priority}                    # 101 on master, 100 on backup
            virtual_ipaddress {
                192.168.1.99
            }
            track_script {
                chk_haproxy
            }
    }" > /etc/keepalived/keepalived.conf

    start keepalived
