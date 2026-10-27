# How to Download Oracle Patches Using `wget.sh`

This document explains how to download Oracle patches from **My Oracle Support (MOS)** using the Oracle-generated `wget.sh` script and an access token.

---

## 1. Download the `wget.sh` Script

1. Log in to **My Oracle Support**.
2. Search for the required patch.
3. Select the required platform.
4. Click **Download**.
5. Click **Download wget Script**.
6. Copy the downloaded `wget.sh` script to the Oracle/OEM server.

Example working directory:

```bash
cd /oracle/oms_bip/Patch
```

Make sure the script exists:

```bash
ls -l wget.sh
```

---

## 2. Generate the Oracle Download Token

Stay logged in to **My Oracle Support** in your browser.

Open another tab in the **same browser session** and go to:

```text
https://updates.oracle.com/Orion/GenToken/get_token
```

Generate the download token.

Then either:

- Click **Copy Token**
- Or download/copy the generated token

The token is a temporary Oracle authentication token used by `wget.sh`.

> Do not use your MOS username or password as the token.

---

## 3. Create a Token File on the Oracle/OEM Server

Create a token file.

The filename can be anything.

Example:

```bash
vi /oracle/oms_bip/Patch/oracle_token.txt
```

Paste **only the generated token** into the file.

Example:

```text
<ORACLE_GENERATED_ACCESS_TOKEN>
```

Do not use:

```text
token=<TOKEN>
```

Do not use:

```text
-t <TOKEN>
```

The file should contain only the token itself.

---

## 4. Protect the Token File

Set secure permissions:

```bash
chmod 600 /oracle/oms_bip/Patch/oracle_token.txt
```

Verify:

```bash
ls -l /oracle/oms_bip/Patch/oracle_token.txt
```

Expected permissions:

```text
-rw------- oracle oinstall oracle_token.txt
```

---

## 5. Verify That the Token File Contains a Usable Token

Use the following command without displaying the actual token:

```bash
awk '
/^[[:space:]]*$/ {next}
/^[[:space:]]*#/ {next}
{print "Usable token found, length =", length($0); exit}
' /oracle/oms_bip/Patch/oracle_token.txt
```

Expected output:

```text
Usable token found, length = <number>
```

If there is no output, the token file is empty or incorrectly formatted.

---

## 6. Configure Proxy If Required

If the server requires an HTTP/HTTPS proxy:

```bash
export https_proxy='https://<PROXY_HOST>:<PORT>'
```

Example:

```bash
export https_proxy='https://proxy.example.com:8086'
```

Verify:

```bash
echo $https_proxy
```

> For a public GitHub repository, do not publish internal company proxy hostnames. Use placeholders such as `<PROXY_HOST>` and `<PORT>`.

---

## 7. Run the Oracle `wget.sh` Script

Run:

```bash
sh wget.sh \
  -T /oracle/oms_bip/Patch/oracle_token.txt
```

The `-T` option tells the script to read the Oracle access token from the specified file.

Example:

```bash
cd /oracle/oms_bip/Patch

sh wget.sh \
  -T /oracle/oms_bip/Patch/oracle_token.txt
```

---

## 8. Specify an Output Directory

If required, use:

```bash
-p <directory>
```

Example:

```bash
sh wget.sh \
  -T /oracle/oms_bip/Patch/oracle_token.txt \
  -p /oracle/oms_bip/Patch
```

If `-p` is not specified, the script uses the current directory.

You may see:

```text
INFO: Output dir not specified, defaulting to current directory.
```

This is informational and not an error.

---

## 9. Verify the Download

Check the downloaded patch:

```bash
ls -lh
```

Example:

```text
p39915792_190000_Linux-x86-64.zip
```

If `unzip` is available, verify the ZIP file:

```bash
unzip -t p39915792_190000_Linux-x86-64.zip
```

---

## 10. Reusing the Token

The same token can generally be used for multiple Oracle patch downloads while it remains valid.

For example:

```bash
sh wget_patch1.sh -T /oracle/oms_bip/Patch/oracle_token.txt
```

```bash
sh wget_patch2.sh -T /oracle/oms_bip/Patch/oracle_token.txt
```

Generate a new token when:

- The current token expires
- The script reports HTTP `401`
- The script reports HTTP `403`
- Oracle reports that the token is invalid

---

## 11. Common Errors

### Missing Token

Error:

```text
ERROR: Missing token. Provide -t <token> OR -T <token_file>.
```

Cause:

The script was run without an Oracle access token.

Solution:

```bash
sh wget.sh \
  -T /oracle/oms_bip/Patch/oracle_token.txt
```

---

### Token File Has No Usable Token

Error:

```text
ERROR: Token file has no usable token line
```

Check:

```bash
wc -c /oracle/oms_bip/Patch/oracle_token.txt
```

Then verify:

```bash
awk '
/^[[:space:]]*$/ {next}
/^[[:space:]]*#/ {next}
{print "Usable token found, length =", length($0); exit}
' /oracle/oms_bip/Patch/oracle_token.txt
```

If there is no output, regenerate the Oracle token and paste only the token into the file.

---

### Token Expired

Possible errors:

```text
HTTP 401
```

or:

```text
HTTP 403
```

Generate a fresh Oracle download token and replace the contents of:

```bash
/oracle/oms_bip/Patch/oracle_token.txt
```

Then rerun:

```bash
sh wget.sh \
  -T /oracle/oms_bip/Patch/oracle_token.txt
```

---

## 12. Security Notes

Never commit the following to GitHub:

```text
oracle_token.txt
Oracle access tokens
MOS passwords
Company proxy credentials
Oracle wallet passwords
Private keys
```

Add the token file to `.gitignore`:

```gitignore
oracle_token.txt
*token*.txt
```

The GitHub repository should contain only documentation and example configuration.

---

# Quick Reference

```bash
# Go to patch directory
cd /oracle/oms_bip/Patch

# Create token file
vi oracle_token.txt

# Protect token
chmod 600 oracle_token.txt

# Configure proxy if required
export https_proxy='https://<PROXY_HOST>:<PORT>'

# Download Oracle patch
sh wget.sh -T /oracle/oms_bip/Patch/oracle_token.txt

# Verify downloaded files
ls -lh
```

---

# Download Flow

```text
My Oracle Support
       |
       v
Search Patch
       |
       v
Download wget Script
       |
       v
wget.sh
       |
       +------------------------+
       |                        |
       v                        v
Generate MOS Token        Copy wget.sh
       |                   to Oracle Server
       v                        |
oracle_token.txt                |
       |                        |
       +-----------+------------+
                   |
                   v
        sh wget.sh -T token_file
                   |
                   v
          Oracle Patch Download
                   |
                   v
               Verify ZIP
```

---

## Important

`wget.sh` and the Oracle download token serve different purposes:

```text
wget.sh
   =
Contains patch download information

Oracle Download Token
   =
Authenticates the download request

SHA-256 Digest
   =
Verifies downloaded patch integrity
```

All three may be involved in the patch download and validation process.
