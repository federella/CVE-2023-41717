# CVE-2023-41717
Inappropriate file type control in Zscaler Proxy versions 3.6.1.25 and prior allows local attackers to bypass file download/upload restrictions.


## Executive Summary
During the summer of 2022, I have found a vulnerability affecting the ZScaler proxy (versions 3.6.1.25 and prior). This vulnerability would allow local attackers to bypass the restriction on downloads/uploads of password-protected archives using tools like Burp or even native Microsoft utilities like Bitsadmin, which relies on the Background Intelligent Transfer Service (BITS) protocol.

Per Microsoft’s documentation, the BITS protocol *“defines a way to transfer large payloads from a client to an HTTP server or vice versa, even in the face of interruptions, by sending the payload in multiple fragments”*. This allows to bypass restrictions based on file type as Zscaler is not able to properly reconstruct the file across multiple requests. 

Although this proof of concept will focus only on the _download_ aspect, this vulnerability applies to the uploads as well.

## Proof of Concept
In this section, two different methods for bypassing Zscaler’s restrictions on downloading password-protected archives are highlighted. 
Tests were performed on version 3.6.1.25 of the client, using the following [URL](https://www.malware-traffic-analysis.net/2022/10/04/2022-10-04-IOCs-for-IcedID-infection-with-Cobalt-Strike.txt.zip). 

### Method 1: Manual crafting of HTTP requests

The first method involves modifying HTTP request, which can be done either using a browser or a tool like Burp Suite. For the sake of this test, I have chosen the former.

The image below shows the request being intercepted and blocked by Zscaler. The request is then retransmitted after adding the `Range` header with value `bytes = 0-x`, where `x` is an arbitrary value smaller than the total file size.

![ZScaler blocking password-protected downloads](https://github.com/federella/CVE-2023-41717/blob/main/images/1.png)

Once the request is retransmitted, a response with status code “206 Partial Content” is received. The response headers will show the total file size, while the response payload is encoded in base64.

![Sample response 1/2](https://github.com/federella/CVE-2023-41717/blob/main/images/2.png)

![Sample response 2/2](https://github.com/federella/CVE-2023-41717/blob/main/images/3.png)

The requests are repeated by manually increasing the byte range value, until the last file chunk is reached.

![Multiple requests](https://github.com/federella/CVE-2023-41717/blob/main/images/4.png)

The resulting payload can be reconstructed in various ways: for this test, a custom Powershell script has been used (you can find it in **Reconstruct-Payload.ps1**). 

The MD5 hash of the reconstructed zip file (`CE6CFFEA60C6CDF40C998E56B6EFBD20`) matches the expected one found on Virus Total.

![PowerShell Script](https://github.com/federella/CVE-2023-41717/blob/main/images/5.png)


![Resulting hash](https://github.com/federella/CVE-2023-41717/blob/main/images/6.png)

![Hash check on VT](https://github.com/federella/CVE-2023-41717/blob/main/images/7.png)

### Method 2: BITS

The second method leverages Microsoft's BITS protocol, which natively splits the download requests in chunks.

This test has been done using the CLI utility **bitsadmin.exe**, with the following command line:

`bitsadmin.exe /transfer <job name> /download /priority normal <URL> <path_destination_file>`

![Bitsadmin commandline](https://github.com/federella/CVE-2023-41717/blob/main/images/8.png)


![Bitsadmin test](https://github.com/federella/CVE-2023-41717/blob/main/images/9.png)

The resulting zip’s MD5 hash matches the one found in the previous section.
