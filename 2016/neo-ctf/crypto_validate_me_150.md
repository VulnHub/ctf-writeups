###Solved by bitvijays


Validate Me
150


-----BEGIN PGP PUBLIC KEY BLOCK-----

mQENBFbp4dgBCAC5XARya41ip+Z4f0kZvGLttOJdP7OkeNFC8AZo/jXDagwrjBIz
eo26g2BAXgNs26llquiHj+J+OXmdBtYE5zUps+4zE1mCkeeE3TxoeK1efsom7nRi
SkPe7/wtRXjd5Eyk7PMWpFfzCo4H0uVXV55Hevxvhjby+D9+jGMagIr+ZuE1lAzz
X61TGBguQFMrK65PRhBS0guIuLjIElcWswATTCBRBYpha3iz/+gFUlk1j5upL8th
hf2SHPI9ND5CBoTm4OIXRNy1+hHHWCU8UCbHuR2ZsCtf9Cw560TZ6P2/REWbm7+d
tMWEPJW7d0CtJJQjBw5+hrj5iXik420jCCqbABEBAAG0PGZsYWd7UHJldHR5X0dv
b2RfUGxhaWRfUGFybGlhbWVudF9PZl9Qd25pbmcgPHBncEBuZW9jdGYuY29tPokB
NwQTAQoAIQUCVunh2AIbAwULCQgHAwUVCgkICwUWAgMBAAIeAQIXgAAKCRAbbdeR
uGaFYF5XB/wImcUfxdyppPBhg2DQjUWUtxXGvL5RueEQ+YwYLr8VlgIj6RLCV8DI
vh3SMm3h5lgG4kbhEETtQGIcREacUzYG1s8XZDLPaTFIs6pEc3Bl2xdSrJkgiZgR
P00Oc/BtjqK0zATuWY2uFXxR8J4mpSAL7it9VhclQRNuF3WSrhjJ7zLeQ1otA8tW
k6c2TtHsTVfaiwhG+sKWWaOxO4znqfkAWSBPgh90wQo5wZl1khpMwX2jylV4oUyu
3CJYidB8duQqAXfdHVejCSs16Ct5XUMSGlj9EYNaahB4Gidq/r+XXTyjCoXvoQsY
aIYuFpN5pHElwm65Xz2sBSiDXphWDvk7uQENBFbp4dgBCADAdt9WaPk70sSoT8jd
RKxiI/pKaOyqk5JU4LyFGDzpnS2LU/o0VGA0UHpKlSTWXV3c/F/6dNx644THvdUX
nryMFl1pT+3lyrSunNs0eYeITP0ts/hnJrDf8xy9qaKFlb+7Wf7BlJYoCmBlt6l4
C9hXX0o5x/e38qqRdiPSlDQg/r71wsjB9nBOqLXDKSFXKhui9OmGmqeIOz2u9mw4
bGHaNdhoEX53CGpUhJRkllOjb7iovu2RyxnQvPv3/S3FBIj33KMx4KFtXCgs/mMB
YmHwx1LlhTjWZnDX1x+b3d9xSPMRqdhpG1Izlpjn/OYvyQc8khcVmwywzodt2oQv
SaAJABEBAAGJAR8EGAEKAAkFAlbp4dgCGwwACgkQG23XkbhmhWDpGQgAqJYGtdU1
UxRuss0I9amdjD/MF5QgyrwpdUa7YBVBOys9ohjEKFFIV0ngsL13HmogshoECCZ+
se1/rr7Qnc635fu+DDlrSRV5cBjH/J/IdhOHqIKBZ+vXwoCCwUL7ljd2qzJhHrlw
+8sxgQBreOhFw5PWU2Vdluy0WCWbRyDIFvo2nUhsF215ciy9ewWzv4hFyiha1K1P
FK7pxc+rUBHtF7C9mDVS2fWw67weREv2QRktwqkqXBCF5VVOkHEdGvTxl8spbZoU
98xHI9ZvisQjURu63ZjoNIO3QbR2+24x8iUwSvFPx0/gJuXuPNue+CbYb9psb+q5
qPAUeskQf4OGjw==
=4M/G
-----END PGP PUBLIC KEY BLOCK----

I had no idea how to validate this, However, started to understand pgp key server and found http://keyserver.pgp.com/vkd/GetWelcomeScreen.event which is the pgp key server stores the public key, came to know about PGP contains Name, email_id and a key id.

Searching for how to extract key id from pgp block, I came across http://security.stackexchange.com/questions/43348/extracting-the-pgp-keyid-from-the-public-key-file  

which says 

```
gpg --dry-run --import pubkey.asc
```
which provided me with the key_id, searching that key_id didn't helped much. So I imported the public_key

```
gpg --import pubkey.asc
gpg: key B8668560: "flag{Pretty_Good_Plaid_Parliament_Of_Pwning <pgp@neoctf.com>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1
```

The flag is **flag{Pretty_Good_Plaid_Parliament_Of_Pwning}**

