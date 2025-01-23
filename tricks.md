# Stratoshark Remote Capture

## Terminal 1

Start a netcat listener to capture the output:

```bash
nc -l 6666 > demo.scap
```

## Terminal 2

SSH into the target machine where you want to capture the trace:

```bash
ssh -i jumpbox.pem ubuntu@15.237.181.68 -R 6666:127.0.0.1:6666
```

Run sysdig to capture the trace and send it to the listener:

```bash
sudo sysdig --unbuffered -w - | nc 127.0.0.1 6666
```

Alternatively, use a filter to exclude specific processes:

```bash
sudo sysdig "proc.name!=sshd and proc.name!=nc" --unbuffered -w - | nc 127.0.0.1 6666
```

## Terminal 1

After terminating the capture, review the file locally using sysdig (if installed):

```bash
sysdig -r ./demo.scap
```

Or open the capture in Stratoshark (on Mac):

```bash
/Applications/Stratoshark.app/Contents/MacOS/Stratoshark ./demo.scap
```