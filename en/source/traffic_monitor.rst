.. _ch_traffic_monitor:

Traffic Monitor
===============

This section describes how to add a function to monitor OpenFlow switch statistical information to the switching hub explained in " :ref:`ch_switching_hub` ".


Routine Examination of Network
------------------------------

Networks have already become the infrastructure of many services and businesses, so maintaining of normal and stable operation is expected. Having said that, problems always occur.

When an error occurred on network, the cause must be identified and operation restored quickly. Needless to say, in order to detect errors and identify causes, it is necessary to understand the network status on a regular basis. For example, assuming the traffic volume of a port of some network device indicates a very high value, whether it is an abnormal state or is usually that way and when it became that way cannot be determined if the port's traffic volume has not been measured continuously.

For this reason, constant monitoring of the health of a network is essential for continuous and safe operation of the services or businesses that use that network.
As a matter of course, simply monitoring traffic information does not provide a perfect guarantee but this section describes how to use OpenFlow to acquire statistical information for a switch.


Implementation of Traffic Monitor
---------------------------------

The following is source code in which a traffic monitoring function has been added to the switching hub explained in " :ref:`ch_switching_hub` ".

.. rst-class:: sourcecode

.. literalinclude:: sources/simple_monitor.py

The traffic monitor function has been implemented in the SimpleMonitor class which inherited SimpleSwitch13, therefore, there is no packet transfer-related processing here.


Fixed-Cycle Processing
^^^^^^^^^^^^^^^^^^^^^^

In parallel with switching hub processing, create a thread to periodically issue a request to the OpenFlow switch to acquire statistical information.

.. rst-class:: sourcecode

::

    from operator import attrgetter
    
    from ryu.app import simple_switch_13
    from ryu.controller import ofp_event
    from ryu.controller.handler import MAIN_DISPATCHER, DEAD_DISPATCHER
    from ryu.controller.handler import set_ev_cls
    from ryu.lib import hub


    class SimpleMonitor(simple_switch_13.SimpleSwitch13):

        def __init__(self, *args, **kwargs):
            super(SimpleMonitor, self).__init__(*args, **kwargs)
            self.datapaths = {}
            self.monitor_thread = hub.spawn(self._monitor)
    # ...

There are some eventlet wrappers and basic class implementation in ``ryu.lib.hub``. Here, we use ``hub.spawn()``, which creates threads.
The thread actually created is an eventlet green thread.

.. rst-class:: sourcecode

::

    # ...
    @set_ev_cls(ofp_event.EventOFPStateChange,
                [MAIN_DISPATCHER, DEAD_DISPATCHER])
    def _state_change_handler(self, ev):
        datapath = ev.datapath
        if ev.state == MAIN_DISPATCHER:
            if not datapath.id in self.datapaths:
                self.logger.debug('register datapath: %016x', datapath.id)
                self.datapaths[datapath.id] = datapath
        elif ev.state == DEAD_DISPATCHER:
            if datapath.id in self.datapaths:
                self.logger.debug('unregister datapath: %016x', datapath.id)
                del self.datapaths[datapath.id]

    def _monitor(self):
        while True:
            for dp in self.datapaths.values():
                self._request_stats(dp)
            hub.sleep(10)
    # ...

In thread function ``_monitor()``, issuance of a statistical information acquisition request for the registered switch is repeated infinitely every 10 seconds.

In order to make sure the connected switch is monitored, ``EventOFPStateChange`` event is used for detecting connection and disconnection. This event is issued by the Ryu framework and is issued when the Datapath state is changed.

Here, when the Datapath state becomes ``MAIN_DISPATCHER``, that switch is registered as the monitor target and when it becomes ``DEAD_DISPATCHER``, the registration is deleted.

.. rst-class:: sourcecode

::

    # ...
    def _request_stats(self, datapath):
        self.logger.debug('send stats request: %016x', datapath.id)
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser

        req = parser.OFPFlowStatsRequest(datapath)
        datapath.send_msg(req)

        req = parser.OFPPortStatsRequest(datapath, 0, ofproto.OFPP_ANY)
        datapath.send_msg(req)
    # ...

With periodically called ``_request_stats()``, ``OFPFlowStatsRequest`` and ``OFPPortStatsRequest`` are issued to the switch.

``OFPFlowStatsRequest`` requests that the switch provide statistical information related to flow entry.
The requested target flow entry can be narrowed down by conditions such as table ID, output port, cookie value and match but here all entries are made subject to the request.

``OFPPortStatsRequest`` request that the switch provide port-related statistical information.
It is possible to specify the desired port number to acquire information from. Here, ``OFPP_ANY`` is specified to request information from all ports.


FlowStats
^^^^^^^^^
In order to receive a response from the switch, create an event handler that receives the FlowStatsReply message.

.. rst-class:: sourcecode

::

    # ...
    @set_ev_cls(ofp_event.EventOFPFlowStatsReply, MAIN_DISPATCHER)
    def _flow_stats_reply_handler(self, ev):
        body = ev.msg.body

        self.logger.info('datapath         '
                         'in-port  eth-dst           '
                         'out-port packets  bytes')
        self.logger.info('---------------- '
                         '-------- ----------------- '
                         '-------- -------- --------')
        for stat in sorted([flow for flow in body if flow.priority == 1],
                           key=lambda flow: (flow.match['in_port'],
                                             flow.match['eth_dst'])):
            self.logger.info('%016x %8x %17s %8x %8d %8d',
                             ev.msg.datapath.id,
                             stat.match['in_port'], stat.match['eth_dst'],
                             stat.instructions[0].actions[0].port,
                             stat.packet_count, stat.byte_count)
    # ...

``OPFFlowStatsReply`` class's attribute ``body`` is the list of ``OFPFlowStats`` and stores the statistical information of each flow entry, which was subject to FlowStatsRequest.

All flow entries are selected except the Table-miss flow of priority 0. The number of packets and bytes matched to the respective flow entry are output by being sorted by the received port and destination MAC address.

Here, only part of values are output to the log but in order to continuously collect and analyze information, linkage with external programs may be required. In such a case, the content of ``OFPFlowStatsReply`` can be converted to the JSON format.

For example, it can be written as follows:

.. rst-class:: sourcecode

::

    import json

    # ...

    self.logger.info('%s', json.dumps(ev.msg.to_jsondict(), ensure_ascii=True,
                                      indent=3, sort_keys=True))

In this case, the output is as follows:

.. rst-class:: console

::

    {
       "OFPFlowStatsReply": {
          "body": [
             {
                "OFPFlowStats": {
                   "byte_count": 0, 
                   "cookie": 0, 
                   "duration_nsec": 680000000, 
                   "duration_sec": 4, 
                   "flags": 0, 
                   "hard_timeout": 0, 
                   "idle_timeout": 0, 
                   "instructions": [
                      {
                         "OFPInstructionActions": {
                            "actions": [
                               {
                                  "OFPActionOutput": {
                                     "len": 16, 
                                     "max_len": 65535, 
                                     "port": 4294967293, 
                                     "type": 0
                                  }
                               }
                            ], 
                            "len": 24, 
                            "type": 4
                         }
                      }
                   ], 
                   "length": 80, 
                   "match": {
                      "OFPMatch": {
                         "length": 4, 
                         "oxm_fields": [], 
                         "type": 1
                      }
                   }, 
                   "packet_count": 0, 
                   "priority": 0, 
                   "table_id": 0
                }
             }, 
             {
                "OFPFlowStats": {
                   "byte_count": 42, 
                   "cookie": 0, 
                   "duration_nsec": 72000000, 
                   "duration_sec": 57, 
                   "flags": 0, 
                   "hard_timeout": 0, 
                   "idle_timeout": 0, 
                   "instructions": [
                      {
                         "OFPInstructionActions": {
                            "actions": [
                               {
                                  "OFPActionOutput": {
                                     "len": 16, 
                                     "max_len": 65509, 
                                     "port": 1, 
                                     "type": 0
                                  }
                               }
                            ], 
                            "len": 24, 
                            "type": 4
                         }
                      }
                   ], 
                   "length": 96, 
                   "match": {
                      "OFPMatch": {
                         "length": 22, 
                         "oxm_fields": [
                            {
                               "OXMTlv": {
                                  "field": "in_port", 
                                  "mask": null, 
                                  "value": 2
                               }
                            }, 
                            {
                               "OXMTlv": {
                                  "field": "eth_dst", 
                                  "mask": null, 
                                  "value": "00:00:00:00:00:01"
                               }
                            }
                         ], 
                         "type": 1
                      }
                   }, 
                   "packet_count": 1, 
                   "priority": 1, 
                   "table_id": 0
                }
             }
          ], 
          "flags": 0, 
          "type": 1
       }
    }


PortStats
^^^^^^^^^

In order to receive a response from the switch, create an event handler that receives the PortStatsReply message.

.. rst-class:: sourcecode

::

    # ...
    @set_ev_cls(ofp_event.EventOFPPortStatsReply, MAIN_DISPATCHER)
    def _port_stats_reply_handler(self, ev):
        body = ev.msg.body

        self.logger.info('datapath         port     '
                         'rx-pkts  rx-bytes rx-error '
                         'tx-pkts  tx-bytes tx-error')
        self.logger.info('---------------- -------- '
                         '-------- -------- -------- '
                         '-------- -------- --------')
        for stat in sorted(body, key=attrgetter('port_no')):
            self.logger.info('%016x %8x %8d %8d %8d %8d %8d %8d', 
                             ev.msg.datapath.id, stat.port_no,
                             stat.rx_packets, stat.rx_bytes, stat.rx_errors,
                             stat.tx_packets, stat.tx_bytes, stat.tx_errors)

``OPFPortStatsReply`` class's attribute ``body`` is the list of ``OFPPortStats``.

``OFPPortStats`` stores statistical information such as port numbers, send/receive packet count, respectively, byte count, drop count, error count, frame error count, overrun count, CRC error count, and collision count.

Here, being sorted by port number, receive packet count, receive byte count, receive error count, send packet count, send byte count, and send error count are output.


Executing Traffic Monitor
-------------------------

Now, let's actually execute this traffic monitor.

First of all, as with " :ref:`ch_switching_hub` ", execute Mininet. Do not forget to set OpenFlow13 for the OpenFlow version.

Next, finally, let's execute the traffic monitor.

controller: c0:

.. rst-class:: console

::

    ryu@ryu-vm:~# ryu-manager --verbose ./simple_monitor.py
    loading app ./simple_monitor.py
    loading app ryu.controller.ofp_handler
    instantiating app ./simple_monitor.py
    instantiating app ryu.controller.ofp_handler
    BRICK SimpleMonitor
      CONSUMES EventOFPStateChange
      CONSUMES EventOFPFlowStatsReply
      CONSUMES EventOFPPortStatsReply
      CONSUMES EventOFPPacketIn
      CONSUMES EventOFPSwitchFeatures
    BRICK ofp_event
      PROVIDES EventOFPStateChange TO {'SimpleMonitor': set(['main', 'dead'])}
      PROVIDES EventOFPFlowStatsReply TO {'SimpleMonitor': set(['main'])}
      PROVIDES EventOFPPortStatsReply TO {'SimpleMonitor': set(['main'])}
      PROVIDES EventOFPPacketIn TO {'SimpleMonitor': set(['main'])}
      PROVIDES EventOFPSwitchFeatures TO {'SimpleMonitor': set(['config'])}
      CONSUMES EventOFPErrorMsg
      CONSUMES EventOFPPortDescStatsReply
      CONSUMES EventOFPHello
      CONSUMES EventOFPEchoRequest
      CONSUMES EventOFPSwitchFeatures
    connected socket:<eventlet.greenio.GreenSocket object at 0x343fb10> address:('127.0.0.1', 55598)
    hello ev <ryu.controller.ofp_event.EventOFPHello object at 0x343fed0>
    move onto config mode
    EVENT ofp_event->SimpleMonitor EventOFPSwitchFeatures
    switch features ev version: 0x4 msg_type 0x6 xid 0x7dd2dc58 OFPSwitchFeatures(auxiliary_id=0,capabilities=71,datapath_id=1,n_buffers=256,n_tables=254)
    move onto main mode
    EVENT ofp_event->SimpleMonitor EventOFPStateChange
    register datapath: 0000000000000001
    send stats request: 0000000000000001
    EVENT ofp_event->SimpleMonitor EventOFPFlowStatsReply
    datapath         in-port  eth-dst           out-port packets  bytes
    ---------------- -------- ----------------- -------- -------- --------
    EVENT ofp_event->SimpleMonitor EventOFPPortStatsReply
    datapath         port     rx-pkts  rx-bytes rx-error tx-pkts  tx-bytes tx-error
    ---------------- -------- -------- -------- -------- -------- -------- --------
    0000000000000001        1        0        0        0        0        0        0
    0000000000000001        2        0        0        0        0        0        0
    0000000000000001        3        0        0        0        0        0        0
    0000000000000001 fffffffe        0        0        0        0        0        0

In " :ref:`ch_switching_hub` ", the SimpleSwitch13 module name (ryu.app.simple_switch_13) was specified for the ryu-manager command. However, the SimpleMonitor file name (./simple_monitor.py) is specified here.

At this point, there is no flow entry (Table-miss flow entry is not displayed) and the count of each port is all 0.

Let's execute ping from host 1 to host 2.

host: h1:

.. rst-class:: console

::

    root@ryu-vm:~# ping -c1 10.0.0.2
    PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
    64 bytes from 10.0.0.2: icmp_req=1 ttl=64 time=94.4 ms

    --- 10.0.0.2 ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 94.489/94.489/94.489/0.000 ms
    root@ryu-vm:~# 

Packet transfer and flow entry are registered and statistical information is changed.

controller: c0:

.. rst-class:: console

::

    datapath         in-port  eth-dst           out-port packets  bytes
    ---------------- -------- ----------------- -------- -------- --------
    0000000000000001        1 00:00:00:00:00:02        2        1       42
    0000000000000001        2 00:00:00:00:00:01        1        2      140
    datapath         port     rx-pkts  rx-bytes rx-error tx-pkts  tx-bytes tx-error
    ---------------- -------- -------- -------- -------- -------- -------- --------
    0000000000000001        1        3      182        0        3      182        0
    0000000000000001        2        3      182        0        3      182        0
    0000000000000001        3        0        0        0        1       42        0
    0000000000000001 fffffffe        0        0        0        1       42        0

According to the flow entry statistical information, traffic matched to the receive port 1's flow is recorded as 1 packet, 42 bytes. With receive port 2, it is 2 packets, 140 bytes.

According to the port statistical information, the receive packet count (rx-pkts) of port 1 is 3, the receive byte count (rx-bytes) is 182 bytes. With port 2, it is 3 packets and 182 bytes, respectively.

Figures do not match between the statistical information of flow entry and that of port. The reason for that is because the flow entry statistical information is the information of packets that match the entry and were transferred. That means, packets for which Packet-In is issued by Table-miss and are transferred by Packet-Out are not included in these statistics.

In this case, three packets that is the ARP request first broadcast by host 1, the ARP reply returned by host 2 to host 1, and the echo request issued by host 1 to host 2, are transferred by Packet-Out.
For the above reason, the amount of port statistics is larger than that of flow entry.


Conclusion
----------

The section described the following items using a statistical information acquisition function as material.

* Thread generation method by Ryu application
* Capturing of Datapath status changes
* How to acquire FlowStats and PortStats
