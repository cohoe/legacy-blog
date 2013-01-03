---
layout: post
title: "Makefile-based PKI Management"
date: 2013-01-01 23:03
comments: true
sharing: true
categories: Guides
---
Using a couple of seed files, you can easily get started managing your own PKI for your network. You need to have OpenSSL and Make installed on the system you are working with. 

First, pick a directory to store your certificates in. I am going to use <code>/etc/pki/example.com</code> since this is where all the other system SSL stuff is stored and for you SELinux people, will already have the right contexts. cd into that directory and create the Makefile:
``` make
CONFIG=config/openssl.cnf
PUB=host-cert.pem
PRIV=host-key.pem
REQ=req.pem
PUBNPRIV=host-combined.pem

issue:
	mkdir -p ${DEST}
	openssl req -new -nodes -out ${DEST}/${REQ} -config ${CONFIG}
	mv ${PRIV} ${DEST}/
	openssl ca -out ${DEST}/${PUB} -config ${CONFIG} -infiles ${DEST}/${REQ}
	if [ ${DEST} = mail -o ${DEST} = mail/ ]; then cp ${DEST}/${PUB} ${DEST}/${PUBNPRIV}; cat ${DEST}/${PRIV} >> ${DEST}/${PUBNPRIV}; fi
	chmod go= ${DEST}/${PRIV}
	if [ -e ${DEST}/${PUBNPRIV} ]; then chmod go= ${DEST}/${PUBNPRIV}; fi

revoke:
	openssl ca -config ${CONFIG} -revoke ${SOURCE}/${PUB}
```
This lets us perform two actions (targets): Issue and Revoke. Issue will create a new host certificate from your CA. Revoke will expire an already-existing certificate. This is mainly used for renewal since you need to revoke then issue.

Next we will need a configuration directory and some basic files:
``` console
[root@gmcsrvx1 example.com ]# mkdir config config/newcerts config/CA
[root@gmcsrvx1 example.com ]# touch config/index.txt
[root@gmcsrvx1 example.com ]# echo '01' > config/serial
```
After that, cd into the config directory and create an <code>openssl.cnf</code> file. This is the OpenSSL configuration file used for certificate generation:
``` text
# OpenSSL configuration file.
#
# Adapted by Grant Cohoe 2011 (www.grantcohoe.com)
# Original by some past CSHer (www.csh.rit.edu)

# Establish working directory.

dir				= /etc/pki/example.com

[ req ]
default_bits		= 2048			# Size of keys
default_keyfile	= host-key.pem		# name of generated keys
default_md		= sha1			# message digest algorithm
string_mask		= nombstr			# permitted characters
distinguished_name	= req_distinguished_name
req_extensions		= v3_req			# always use v3 extensions for reqs

[ req_distinguished_name ]
# Variable name  Prompt string
#----------------------  ----------------------------------
0.organizationName		= Organization Name (company)
organizationalUnitName	= Organizational Unit Name (department, division)
emailAddress			= Email
emailAddress_max		= 40
localityName			= Locality Name (city)
stateOrProvinceName		= State or Province Name (full name)
countryName			= Country Name (2 letter code)
countryName_min		= 2
countryName_max		= 2
commonName			= Common Name (full hostname)
commonName_max			= 64

# Default values for the above, for consistency and less typing.
# Variable name  Value
#------------------------------  ------------------------------
0.organizationName_default		= Example Corp
organizationalUnitName_default	= System Engineering
emailAddress_default			= root@example.com
localityName_default			= Rochester
stateOrProvinceName_default		= New York
countryName_default				= US

[ v3_ca ]
basicConstraints		= CA:TRUE
subjectKeyIdentifier	= hash
authorityKeyIdentifier	= keyid:always,issuer:always

[ v3_req ]
basicConstraints		= CA:FALSE
subjectKeyIdentifier	= hash


[ ca ]
default_ca	= CA_default

[ CA_default ]
serial		= $dir/serial
database		= $dir/index.txt
new_certs_dir	= $dir/newcerts
certificate	= $dir/CA/example-ca.crt
private_key	= $dir/CA/example-ca.key
default_days	= 365
default_md	= sha1
preserve		= no
email_in_dn	= no
nameopt		= default_ca
certopt		= default_ca
policy		= policy_match

[ policy_match ]
countryName			= match
stateOrProvinceName		= match
organizationName		= match
organizationalUnitName	= optional
commonName			= supplied
emailAddress			= optional
```
You should ensure that the directory definition and organization information is changed to match what you want. 

After this you are ready to generate your CA:
``` console
openssl req -config openssl.cnf -new -x509 -extensions v3_ca -keyout CA/example-ca.key -out CA/example-ca.crt -days 1825
```
This will generate a CA certificate based on your settings file and will last for 5 years, after which you will need to renew it. 

Enter a certificate password when prompted. You do not need to specify a Common Name (CN) as asked later in the process. Ensure that the other field defaults are correct, as they came from your <code>openssl.cnf</code> file and are needed to match in other certificates. 
``` console
[root@gmcsrvx1 config]# openssl req -config openssl.cnf -new -x509 -extensions v3_ca -keyout CA/example-ca.key -out CA/example-ca.crt -days 1825
Generating a 2048 bit RSA private key
................................................................................
..................+++
writing new private key to 'CA/example-ca.key'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Organization Name (company) [Example Corp]:
Organizational Unit Name (department, division) [System Engineering]:
Email [root@example.com]:
Locality Name (city) [Rochester]:
State or Province Name (full name) [New York]:
Country Name (2 letter code) [US]:
Common Name (full hostname) []:
[root@gmcsrvx1 config]#
```

Unfortunately this process creates the private key file to be world readable. This is extremely bad. Fix it. Also ensure that the public key is readable by all. It is supposed to be.
``` console
[root@gmcsrvx1 config]# chmod 600 CA/example-ca.key
[root@gmcsrvx1 config]# chmod 644 CA/example-ca.crt
```

Now back up one directory (back to the working directory you started at). I usually make a symlink to the CA public key here so you can easily reference it. 

You can now issue SSL certificates by typing <code>make DEST=hostname</code> where hostname is the name of the server you are going to issue to. 
``` console
[root@gmcsrvx1 example.com]# make DEST=gmcsrvx1
mkdir -p gmcsrvx1
openssl req -new -nodes -out gmcsrvx1/req.pem -config config/openssl.cnf
Generating a 2048 bit RSA private key
..............................................+++
......................................+++
writing new private key to 'host-key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Organization Name (company) [Example Corp]:
Organizational Unit Name (department, division) [System Engineering]:
Email [root@example.com]:
Locality Name (city) [Rochester]:
State or Province Name (full name) [New York]:
Country Name (2 letter code) [US]:
Common Name (full hostname) []:gmcsrvx1.example.com
mv host-key.pem gmcsrvx1/
openssl ca -out gmcsrvx1/host-cert.pem -config config/openssl.cnf -infiles gmcsr
Using configuration from config/openssl.cnf
Enter pass phrase for /etc/pki/example.com/config/CA/example-ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
organizationName      :PRINTABLE:'Example Corp'
organizationalUnitName:PRINTABLE:'System Engineering'
localityName          :PRINTABLE:'Rochester'
stateOrProvinceName   :PRINTABLE:'New York'
countryName           :PRINTABLE:'US'
commonName            :PRINTABLE:'gmcsrvx1.example.com'
Certificate is to be certified until May  1 18:12:03 2013 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
if [ gmcsrvx1 = mail -o gmcsrvx1 = mail/ ]; then cp gmcsrvx1/host-cert.pem gmcsr.pem; fi
chmod go= gmcsrvx1/host-key.pem
if [ -e gmcsrvx1/host-combined.pem ]; then chmod go= gmcsrvx1/host-combined.pem;
[root@gmcsrvx1 example.com]#
```

If you look in the directory it made for you (the hostname you specified) you will find the private and public keys for the server.
``` console
[root@gmcsrvx1 example.com]# ls -l gmcsrvx1/
total 12
-rw-r--r--. 1 root root 3997 May  1 14:12 host-cert.pem
-rw-------. 1 root root 1704 May  1 14:12 host-key.pem
-rw-r--r--. 1 root root 1171 May  1 14:12 req.pem
```

You can revoke this certificate using the makefile as well:
``` console
[root@gmcsrvx1 example.com]# make revoke SOURCE=gmcsrvx1
openssl ca -config config/openssl.cnf -revoke gmcsrvx1/host-cert.pem
Using configuration from config/openssl.cnf
Enter pass phrase for /etc/pki/example.com/config/CA/example-ca.key:
Revoking Certificate 01.
Data Base Updated
[root@gmcsrvx1 example.com]#
```

There you go, yet another useful service to have around for dealing with your network. Raw files used above are available in the <a href="http://archive.grantcohoe.com/projects/ssl">Archive</a>. Now back to Webauth with me...
