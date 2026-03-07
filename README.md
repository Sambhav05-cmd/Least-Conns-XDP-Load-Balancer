# XDP Least-Connections Load Balancer

This project implements a NAT-based TCP load balancer using eBPF at the XDP layer.  
It distributes incoming connections across backend servers using the least-connections scheduling algorithm.

Running the load balancer at XDP (eXpress Data Path) allows packets to be processed before entering the Linux networking stack. This removes a large portion of the traditional networking overhead and allows packets to be handled with very low latency.

---

## Overview

The load balancer performs connection-aware traffic distribution in the following way.

Each incoming TCP connection is assigned to the backend server that currently has the lowest number of active connections. The XDP program keeps track of connection state by observing TCP flags and maintaining lightweight connection tracking structures in eBPF maps.

Packet processing happens entirely inside an XDP eBPF program. The program inspects packets immediately when they arrive on the network interface and performs NAT-based redirection to the selected backend server.

Because the load balancer runs at the XDP layer, packets are processed before entering the normal Linux networking stack. This significantly reduces overhead and allows the system to scale to high packet rates with minimal CPU usage.

---
## Two Connection Tracking Modes

This repository provides two connection tracking modes for the least-connections algorithm. Both use the same scheduling logic but differ in when a connection is counted.

In the first mode, a connection is counted only after the TCP handshake completes (when the first non-SYN packet is observed). This ensures the counters reflect only fully established connections. However, when many connections start simultaneously, several SYN packets may see the same backend counters before they update, which can temporarily skew the distribution.

In the second mode, the counter is incremented as soon as a SYN packet arrives. This effectively reserves the backend at the start of the handshake, which results in more accurate load balancing during bursts of concurrent connections. The tradeoff is that connections that never complete the handshake may be briefly counted until they are cleaned up.
---

## Configuration

Initial backend servers are defined in the file:

configs/backends.json

Example configuration:

{
  "backends": [
    "10.0.0.2",
    "10.0.0.3"
  ]
}

You can edit this file and add the IP addresses of your backend servers.  
Backends can also be added or removed dynamically while the load balancer is running through CLI commands.

---

## Running the Load Balancer

First install the required dependencies:

sudo ./scripts/llvm.sh

From the repository root run:

go generate ./cmd/lb
go build -o lb ./cmd/lb
sudo ./lb -i <network-interface> -config configs/backends.json

Example:

sudo ./lb -i wlo1 -config configs/backends.json

---

## Runtime CLI Commands

After starting the program an interactive CLI becomes available:

lb>

The following commands can be used:

add <ip>     Add a backend server  
del <ip>     Remove a backend server  
list         List current backends and connection counts  

Example usage:

lb> add 10.0.0.4  
lb> del 10.0.0.3  
lb> list  

The list command displays the backend servers currently registered along with the number of active connections each backend has.

---

## Observing eBPF Programs

You can verify that the XDP program is attached using:

sudo bpftool prog show

This allows you to confirm that the eBPF program is successfully loaded and attached to the network interface.

---

## Testing the Load Balancer

### Start backend servers

On each backend machine run:

python3 -m http.server 8000

---

### Send requests to the load balancer

From a client machine you can send requests using curl:

curl -v --http1.1 http://<load-balancer-ip>:8000

Using HTTP/1.1 keeps the connection open for a short period of time, which makes it easier to observe the behavior of the connection counters.

---

## Testing under high connection load(multiple connection requests simultaneously)

To simulate a large number of simultaneous clients you can run:

seq 100 | xargs -n1 -P100 -I{} curl -s --http1.1 http://10.45.179.173:8000 > /dev/null

This command launches one hundred curl requests in parallel and sends them to the load balancer.

---

## Checking Kernel TCP Connections

While the test is running you can check the number of active TCP connections with:

ss -tan '( sport = :8000)' | wc -l

This command counts all TCP sockets currently using port 8000.

---

## Observing Backend Distribution

While connections are active you can type the following command inside the load balancer CLI:

list

This prints the number of connections currently assigned to each backend server. It allows you to directly observe how the least-connections algorithm distributes traffic.

If many connections start at the same time you may notice that the established-counting version can briefly produce uneven distribution. The SYN-counting version typically handles large bursts of concurrent connections more evenly because it increments the counters immediately when SYN packets arrive.

---

## Customization

The current implementation balances traffic only for TCP port 8000.

This can be changed directly in the eBPF program located at:

bpf/lb.c

By modifying the port filter the load balancer can be adapted to handle other services.

---

## Repository Structure

bpf/            eBPF/XDP load balancer program  
cmd/lb/         Go user-space loader and CLI  
configs/        Backend configuration file  
scripts/        Utility scripts  

---

## Technologies Used

eBPF  
XDP (eXpress Data Path)  
Go  
Linux networking

---

## Notes

Root privileges are required to attach XDP programs.

The system must support eBPF and XDP. A modern Linux kernel is recommended.

---

## References

Teodor Podobnik – XDP Load Balancer Tutorial  
https://labs.iximiuz.com/tutorials/xdp-load-balancer-700a1d74

iximiuz Labs – Practical Linux networking and eBPF tutorials  
https://labs.iximiuz.com/
