## 1. Install Client Tools

#### `cfssl` and `cfssljson`

Large part of this tutorial is about managing various certificates and we'll use a tool called `cfssl` which is desined for signing, verifying, and bundling of TLS certificates. There is a corresponding `cfssljson` that helps us manipulate the JSON output of `cfssl`. 

For go 1.18 and above you can install them both in one go :)

```
go install github.com/cloudflare/cfssl/cmd/...@latest 
```

#### `kubectl`

This tool allows you to run requests against the kubernetes apiserver.

Check the kubectl [link](https://kubernetes.io/docs/tasks/tools/) for instructions on how to install it for your operating system. 


## 2. Prepare the Raspberry Pi

The easiest way to flash an OS onto a storage medium (SD card) to then run your raspberry pi is to use the 
[Raspberry Pi Imager](https://www.raspberrypi.com/software/).

In the following docs we assume that Ubuntu ≥23 has been flashed.