### Use Case 1: Detection of Aggressive Signups ###   
   
Web-account abuse attack influences the network security significantly. Back to 2007, millions of botnet email accounts were created to send spam emails in a short period from majof Web email service providers. Since there is a limitation of the number of sending emails each day set by email service providers like Hotmail, a spammer will try to sign up accounts as many as possible. Therefore the detection of aggresive signups becomes the most fundamental method.   
   
Based on the premise that normally the signup actions can not happen frequently at a single IP address, a sudden increase of signup activities with a single source IP address is suspicious. We can consider the IP address may be associated with a bot. The author uses a simple EWMA (Exponentially Weighted Moving Average) algorithm to detect sudden changes in signup activities. The first step of implement this algorithm is to calculate the number of IP packets with same source IP addresses in a fixed time duration.   
   
Pseudo code:   
   
```shell
FILE_NAME
SPLIT_TIME # time window in which same source IP addresses should be aggregated
START_TIME # record the start time of program

ipaggcreate -s $FILE_NAME --split-time=${SPLIT_TIME}s -o /tmp/res-${SPLIT_TIME}sec-%d
# -s: aggregate the source IP addresses
# --split-time: start a new output file every time period
# -o: define the format of output file

FILE_NUM # Calculate the number of temporary ouput files which will be used for the loop

# Read the temporary files and sort
for i in $( seq 1 $FILE_NUM )
do
	# Remove the first few lines of temporary output files, sort by the number of IP addresses
	# Remove the temporary output files
done

ELPASED_TIME # Use $START_TIME to calculate the elapsed time
```   
   
As for the parallelization, since `ipaggcreate` can split the trace file based on the split time users set, we can simply add `parallel --pipe` to the `sort` in the loop in pseudo code. And GNU Parallel can utilize the CPU automatically.   

---   

### Use Case 2: Blackholing at IXPs ###   
   
Distributed Denial of Service (DDoS) attack is still a serious threat to the Internet. And the core peering links which the DDoS attack will congest at Internet Exchange Points (IXPs) are also influenced. At present, the main DDoS mitigation method is to blackhole traffic to a specific IP prefix at upstream providers. When an AS is attacked, the victim AS will announce the attacked destination IP prefix upstream network via BGP. And the traffic towards these IP prefixes will be dropped.   
   
For example, Figure 1 depicts how blackholing works. In part (A), both AS2 and AS3 have legitimate traffic towards AS1, however, AS3 traffic contains significant amounts of DDoS traffic as well. Now AS1's IXP-connected router advertises the attacked prefix - usually a more specific - for blackholing towards the route server. All connected members receive the BGP update and choose the blackholing IP address as best next hop address since it is more specific. Therefore not only DDoS attack traffic but also legitimate traffic will be sent to the blackholing IP synchronously.   
   
So the attacked IP prefix can be obtained by aggregating the destination addresses.   

Pseudo code:   
   
```shell
FILE_NAME
START_TIME # record the start time of program

ipsumdump -r $FILE_NAME -d|\ # Read file and output destination IP addresses
grep | cut | sort | uniq | sort # Text formatting; aggregation and sort by destination IP addresses

ELPASED_TIME # Use $START_TIME to calculate the elapsed time
```   
The most troublesome problem for parallelization in this use case is how to split the file. Even though GNU Parallel makes it possible to split the file by the number of lines that users set, a feasible way of recombining the results of split files is still hard to find out. An specific IP prefix has different counts in different output files. My guess is using `awk` to check the IP addresses line by line and then add the counts up. But in this way the time complexity will not be satisfied.   
   
---   
   
### Use Case 3: Online Botnet Detection ###   
   
The increasing development speed of botnet makes it a severe threat to Internet security. The botmaster disseminates bot programs to control tremendous amount of hosts and communicates with these bot hosts through Command and Control (C&C) channel to issue orders. And these hosts compose the botnet. The botmaster can take different types of attacks, e.g., DDoS attack, spamming email attack, information theft, etc.   
   
Distributed botnet has no centralized C&C servers hence bot hosts will connect with their neighbour hosts frequently and exchange keep-alive message when the botmaster annouces an order or update. In distributed botnet, each bot host maintains a list of neighbour hosts and tries to connect with a same group of hosts. Furthermore, the flow order of connecting with other hosts in the list is always fixed which means one flow appears, another flow appears afterwards. These flows has an one-by-one relationship which is called association relationship. Compared to bot hosts the legitimate hosts' behaviour is much more random. In other words, legitimate flows have no association relationship.   
   
The main idea is to extract the association relationship of flows in order to recognize C&C traffic and finally detect the bot hosts.   
   
A TCP flow starts from a three-way handshake and ends with a four-way handshake. All the packets inside this process represent a flow. Since there is a mass of traffic in network, it is not practical to check the association relationship for each pair of flows. Therefore we only inspect flows with the difference number of occurrences less than a threshold nTh (nTh=1 in the experiment) during a same window size. To extract the association relationship of flows, there are 4 jobs to complete. I only consider Job 1 in this use case.   
   
Job1 calculates the occurrence number of flows. Only 3 attributes should be taken into account in this case, source IP address, destination IP address and start time.   
   
Pseudo code:   
   
```shell
FILE_NAME
tcpdump | sort | uniq | sort
	# Check the packet type, e.g., SYN, SYN ACK, FIN ... to recognize a TCP flow and record 
	# both source IP address and destination IP address as well as the time stamp
	# Split the file based on a time window and aggregate the flows
	# Check the difference number of occurrence of each flow and print the flows with difference number 
	# equals to 0
```   

This use case is similar to use case 1 and we could simply use `parallel` after the file split to implement parallelization and utilize the CPU.   
   
### Use Case 4: Congested Link Detection ###   
   
Tracking Packets' Paths and Latency via INT(In-band Network Telemetry)   
   
INT is a framework designed to allow the collection and reporting of network state, by the data plane, without requiring intervention or work by the control plane. An INT switch will add switch metadata to a packet whenever it passes through. At the end of the INT switch in the whole path, the switch will extract the metadata and pass the orginal packet to the end-host. The remote monitoring engine supervises the source INT switches which are at both the beginning and end of the transmission path. Packets contain header fields that are interpreted as "telemetry instructions" by network devices. These instructions tell INT-capable devices what to collect; collected data is written into the packets themselves. Examples of per-packet metadata one can collect via INT include switch IDs, input/outpu port IDs, hop latency, queue occupancy, arrival/departure timestamp, etc.   


Path and Latency Tracking(PLT) is just an application of INT. And PLT is used to collect the physical path and hop latencies for every packet; INT Sinks export only meaningful records to a remote monitoring system as mirrored packets.   
   
PLT can make real-time anomaly detection and alert generation including,   
- Congested connections
- Congestion at switches
- Unused (dead) links and switches
- Imbalance of link utilization (ECMP and LAG)
- Loops
   
So with PLT, we can identify connections affected by planned maintenance or un-planned events and detect hop-latency increases and identify the victims.   
For example, if we plan to check all the hop latencies in the network we can collect the mirrored packets from the INT sinks and aggregate the latencies based on switch IDs.   
   
Pseudo code:   
   
Since we don't have INT capable devices which means we can't collect metadata and it's difficult to test this use case. So pseudo code is not written here, however, PLT monitoring engine contains Spark for streaming and batch data processing which means this use case is indeed practical.   

### Use Case 5: The MINDS - Minnesota Instrusion Detection System ###   
   
Minnesota Intrusion Detection System (MINDS) uses a suite of data mining techniques to automatically detect attacks against computer networks and systems. There are two aspects which MINDS concentrate on so far. Firstly, MINDS can detect new and previously unknown types of intrusions, which often indicate emerging threats. It uses an anomaly detection algorithm that assigns a score to each connection based on its probability of being an intrusion. Secondly, MINDS also shows how association pattern analysis can be used to summarize and characterize anomalous network connections. Given the very high volume of connections obaserved per unit time, such characterization of novel attacks is essential in enabling a security analyst to understand emerging threats.   
   
Figure illustates the process of analyzing real network traffic data using the system. The analyst uses MINDS to analyze these 10-minute data files in a batch mode. The reason the system is running in a batch mode is not due to the time it takes to analyze these files, but it is convenient for the analyst to do so. Before data is fed into the anomaly detecion module, a data filtering step is performed by the analyst to remove network traffic that the analyst is not interested in analyzing.   
   
The first step in MINDS is extracting features that are used in the data mining analysis. Derived features include time-window and connection-windows based features. Let's take time-window based features as an example. They are constructed to capture connections with similar characteristics in the last T seconds. The table shows several time-window based features.   
   
> Count-dest: Number of flows to unique destination IP addresses inside the network in the last T seconds from the same source.   
   
Pseudo code:   
   
```shell
# Split trace file based on time window set by user 
editcap -i $SPLIT_TIME $FILE_NAME /tmp/split_minds.pcap -F libpcap

# Process each piece
for TEMPFILE in `ls /tmp/ | grep -P "^split_minds"`
do
# List both source and destination address
	ipsumdump /tmp/$TEMPFILE -o /tmp/$TEMPFILE.res -sd 

# Remove useless lines | sort by source address and destination address afterwards
	sed '/^!/d;/^-/d' /tmp/$TEMPFILE.res | sort -n -k1,1 -k2,2 |\
# Remove & count the duplicated lines and sort
	uniq -c | sort -nr > $TEMPFILE.result

# Cleanup
	rm /tmp/$TEMPFILE
	rm /tmp/$TEMPFILE.res
done
```
   
During the period of processing the trace file in this use case  we need to split trace file based on time window. Therefore, we can use `Parallel` to realize parallelization similar to Use Case 1.   
   

### Data set discussion: ###   
   
The data set I used in this study comes from http://mawi.wide.ad.jp/mawi/   

##### WIDE MAWI WorkingGroup #####   
> The MAWI (Measurement and Analysis on the WIDE Internet) Working Group is a working group that has carried out network traffic measurement, analysis, evaluation, and verification from the beginning of the WIDE Project. The WIDE Project carries out research activities through the use of the actual network, but simply operating the network alone does not qualify as research. Our goals are to make use of the knowledge gained through network operations, and to evaluate research results under the actual traffic. In other words, we evaluate whether the network behaves as it was designed, and learn from unexpected behaviors. Furthermore, if there is a problem, we seek out the cause and devise measures to solve the problem. However, it is very difficult to perform measurements at the site of operation, especially when in a trouble. Therefore, the MAWI Working Group was created to specialize in measurement and evaluation. Additionally, because some form of measurement is often required in any research, the MAWI WG plays a role to share measurement and analysis information across all working groups.   
   
The packet trace should be universal and representative. At the same time, it should also cover different protocols. MAWI captured all the traces from WIDE backbone. As we check the protocol breakdown of the packet trace, we can see that the trace includes packets of almost all kinds of common protocols. The IPv4 packets make up a large proportion of the whole trace and there are more than 50% packets of TCP protocol and some UDP/ICMP packets inside as well. The file size is 4344.48 MB and the number of packets is more than 64 million which is considerable enough to be taken as the data set. As for the privacy, the group also scrambles the addresses in order to provide anonymity.   
Therefore I choosed the packet trace from MAWI Working Group.   
   
### `ipsumdump` & `tcpdump` comparison: ###   
   
Firstly I tried to use simple 'tcpdump' to aggregate the source IP addresses for the second use case. As we all know that 'tcpdump' is a conventional tool of printing out a description of the contents of packets on a network interface as well as reading from a trace file. Afterwards, I found a new linux tool called 'ipsumdump' which is specially used to aggregate IP addresses. Therefore, I wrote a script to compare their performance in order to know which one is better.   
The setup of experiment is quite simple:   
- Task: obtain the source IP addresses from the same trace file   
- File Size: 740 MB   
- Processor: Intel Core i7-4850HQ CPU @ 2.3 GHz
- Core Number: 1   
- Memory: 2 GB   
- System: Ubuntu 16.04 LTS   
   
Script:   
```shell
#!/bin/bash

SUM_TIME=0
REP_TIME=5
while [ ${REP_TIME} -gt 0  ]
do
    START_TIME=$(date +%s.%N)
    tcpdump -nnr ${1} | cut -d ' ' -f3 > /dev/null
    ELAPSED_TIME=$(echo "$(date +%s.%N)-${START_TIME}" | bc)
    SUM_TIME=$(echo "${SUM_TIME}+${ELAPSED_TIME}" | bc)
    REP_TIME=$((${REP_TIME}-1))
done
AVERAGE_TIME_TCPDUMP=$(echo "scale=5; ${SUM_TIME}/5" | bc)
echo "Average runtime of 'tcpdump' is ${AVERAGE_TIME_TCPDUMP} seconds." >> output

SUM_TIME=0
REP_TIME=5
while [ ${REP_TIME} -gt 0  ]
do
    START_TIME=$(date +%s.%N)
    ipsumdump -r ${1} -s > /dev/null
    ELAPSED_TIME=$(echo "$(date +%s.%N)-${START_TIME}" | bc)
    SUM_TIME=$(echo "${SUM_TIME}+${ELAPSED_TIME}" | bc)
    REP_TIME=$((${REP_TIME}-1))
done
AVERAGE_TIME_IPSUMDUMP=$(echo "scale=5; ${SUM_TIME}/5" | bc)
echo "Average runtime of 'ipsumdump' is ${AVERAGE_TIME_IPSUMDUMP} seconds." >> output

echo "'ipsumdump' is $(echo "scale=4; (${AVERAGE_TIME_TCPDUMP}-${AVERAGE_TIME_IPSUMDUMP})/${AVERAGE_TIME_TCPDUMP}*100" | bc)% faster than 'tcpdump'. " >> output
```   

The command line window looks like: (figure)   
The result indicates that `ipsumdump` is 96.6% faster than `tcpdump` in efficiency. So I chose 'ipsumdump' as the tool of aggregating the IP addresses.   
