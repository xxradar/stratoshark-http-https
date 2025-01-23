# Analyzing HTTP/HTTPS Traffic with Stratoshark

<p align="center">
  <img src="./images/stratoshark_logo_240.png" alt="Logo trace" title="Logo" />
</p>

## Prerequisites
- Ubuntu 24.04 on AWS
- Docker 
- Sysdig CLI 
```
curl -s https://s3.amazonaws.com/download.draios.com/stable/install-sysdig | sudo bash
```
## Obtaining System Call Trace Using Sysdig CLI

First, SSH into the remote host:
```sh
ssh ubuntu@remote-host
```

Inside the terminal, run the following command to filter and capture all `curl` processes on the host:
```sh
sudo sysdig proc.name==curl -s 5000 -w docker-curl-https.scap
```
This command will save the captured data into a file named `docker-curl-https.scap`. The `-s 5000` option sets the snaplen to 5000 bytes, which is the maximum amount of data to capture per system call.

## Running `curl` inside a Docker Container

SSH into the remote host from a different terminal:
```sh
ssh ubuntu@remote-host
```

Run the following Docker command to start a container and execute a `curl` request:
```sh
docker run -t xxradar/hackon curl -L http://www.radarhack.com
```
This command will pull the `xxradar/hackon` image and run a `curl` command to fetch the specified URL. Note that our Sysdig CLI will only trace the `curl` process. If you need more data, you can modify the filter accordingly. <br>
The `curl` process inside the container will connect in cleartext to HTTP port 80 on the remote server and follow the redirect to the actual webpage using TLS and HTTPS on port 443.

## Obtaining the Sysdig Trace

Stop the trace using `Ctrl+C`. Optionally, you can also create a text version of the trace:
```sh
sudo sysdig -r docker-curl-https.scap > docker-curl-https.txt
```
This command reads the captured trace file and outputs it in a human-readable text format.

To transfer the trace files to your local machine, use the following `scp` commands:
```sh
scp ubuntu@remote-host:~/docker-curl-https.scap ./docker-curl-https.scap
scp ubuntu@remote-host:~/docker-curl-https.txt ./docker-curl-https.txt
```
These commands will securely copy the trace files from the remote host to your local machine.<br>
If you want to follow along with the examples in the next section, you can find the trace in this repo in the `traces` directory.

## Stratoshark
The `docker-curl-https.scap` can be opened with Stratoshark, available for Mac and Windows.<br> 
(see https://www.wireshark.org/download/automated/ for downlodad)


From the UI open `docker-curl-https.scap`
![Unfiltered trace](./images/unfiltered_1.png "Unfiltered traces")



### Applying a filter (HTTP part)
As you can see there are a lot of entries, so let's filter in popular `wireshark-style` and nail it down to what we are looking for.
Apply following filter
```
evt.type==connect or evt.type==recvfrom or evt.type==sendto
```
![filtered trace](./images/filtered.png "Filtered traces")

If we look carefully in `line 2395` we find the request being send.
```
2395 21:07:53.946614329 0 curl (104155) < sendto res=81 data=GET / HTTP/1.1..Host: www.radarhack.com..User-Agent: curl/7.81.0..Accept: */*...
```
In `line 2510` we can find the response
```
2510 21:07:53.972731198 0 curl (104155) < recvfrom res=748 data=HTTP/1.1 301 Moved Permanently..Date: Wed, 23 Jan 2025 10:06 GMT..Content-...
```
There is an easier way to track this all down. Select `line 2510`, click right and select `Follow -> File Descriptor Stream` and Stratoshark will do the hard work for you.
![follow_stream_2 trace](./images/follow_stream_2.png "Filtered traces")
The filter is updated accordingly.
![follow_stream_1 trace](./images/follow_stream_1.png "Filtered traces")




### Applying a filter (HTTPS part)

For HTTPS, this is slightly different
```
evt.type==connect or evt.type==read or evt.type=write
```
![tls_1 trace](./images/tls_1.png "tls_1 traces")
![tls_2 trace](./images/tls_2.png "tls_2 traces")
![tls_3 trace](./images/tls_3.png "tls_3 traces")
