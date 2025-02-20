```
set ns[ new Simulation]
set f [open out.tr w]
set nf [open out.nam w]

$ns trace-all $f
$ns namtrace-all $nf

set n0 [$ns node];
set n1 [$ns node];
set n2 [$ns node];
set n3 [$ns node];

$ns duplex-link $n0 $n1 1Mb 10ms DropTail
$ns duplex-link $n0 $n2 1Mb 10ms Droptail
$ns duplex-link $n0 $n3 1Mb 10ms Droptail

set tcp1 [new Agent/TCP]
$ns attach-agent $n1 $tcp1
set sink1 [new Agent/TCPSink]
$ns attach-agent $n1 $sink1
$ns connect $tcp1 $sink1

set tcp2 [new Agent/TCP]
$ns attach-agent $n2 $tcp2
set sink2 [new Agent/TCPSink]
$ns attach-agent $n2 $ sink2
$ns connect $tcp2 $sink2

set tcp3 [new Agent/TCP]
$ns attach-agent $n3 $tcp3
set sink3 [new Agent/TCPSink]
$ns attach-agent $n3 $ sink3
$ns connect $tcp3 $sink3

// traffic

set ftp1 [new Application/FTP]
$ftp1 attach-agent $tcp1
$ftp1 set type_FTP

set ftp2 [new Application/FTP]
$ftp2 attach-agent $tcp2
$ftp2 set type_FTP

set ftp3 [new Application/FTP]
$ftp3 attach-agent $tcp3
$ftp3 set type_FTP

$ns at 0.2 "$ftp1 start"
$ns at 0.4 "$ftp2 start"
$ns at 0.6 "$ftp3 start"

$ns at 10.0 "finish"

proc finish{}{
global ns f nf
$ns flush-trace
exit 0
}

$ns run

```

```
set udp1 [new Agent/UDP]
$ns attach-agent $n0 $udp1
set null[new Agent/Null]
$ns attach-agent $n1 $null0
$ns connect $udp0 $null0

set cbr0 [new Application/Traffic/CBR]
$cbr0 set packetSize_ 500
$cbr0 set interval_ 0.005
$cbr0 attach-agent $udp0
```


```
# Create a simulator instance
set ns [new Simulator]

# Define tracing files
set tracefile [open star_topology.tr w]
$ns trace-all $tracefile

set namfile [open star_topology.nam w]
$ns namtrace-all $namfile

# Create nodes
set hub [$ns node]    ;# Central node (acts as a router)
set n1 [$ns node]
set n2 [$ns node]
set n3 [$ns node]
set n4 [$ns node]
set receiver [$ns node]

# Define Links: Star topology (all through the hub)
$ns duplex-link $n1 $hub 1Mb 10ms DropTail
$ns duplex-link $n2 $hub 1Mb 10ms DropTail
$ns duplex-link $n3 $hub 1Mb 10ms DropTail
$ns duplex-link $n4 $hub 1Mb 10ms DropTail
$ns duplex-link $hub $receiver 2Mb 5ms DropTail

# Set queue size
$ns queue-limit $hub $receiver 20

# Define UDP Traffic
for {set i 1} {$i <= 4} {incr i} {
    set udp($i) [new Agent/UDP]
    set null($i) [new Agent/Null]
    
    # Attach UDP agent to nodes n1-n4
    $ns attach-agent [set n$i] $udp($i)
    
    # Attach Null agent to receiver
    $ns attach-agent $receiver $null($i)

    # Connect UDP agents to Null agents
    $ns connect $udp($i) $null($i)

    # Set CBR traffic over UDP
    set cbr($i) [new Application/Traffic/CBR]
    $cbr($i) set packetSize_ 512
    $cbr($i) set rate_ 100kb
    $cbr($i) set random_ false
    $cbr($i) attach-agent $udp($i)
}

# Start traffic at different times
$ns at 0.5 "$cbr(1) start"
$ns at 1.0 "$cbr(2) start"
$ns at 1.5 "$cbr(3) start"
$ns at 2.0 "$cbr(4) start"

# Stop traffic
$ns at 5.0 "$cbr(1) stop"
$ns at 5.5 "$cbr(2) stop"
$ns at 6.0 "$cbr(3) stop"
$ns at 6.5 "$cbr(4) stop"

# Simulation end
$ns at 7.0 "finish"

proc finish {} {
    global ns tracefile namfile
    $ns flush-trace
    close $tracefile
    close $namfile
    exec nam star_topology.nam &
    exit 0
}

$ns run

```
