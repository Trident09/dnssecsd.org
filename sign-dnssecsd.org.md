DNSSEC Signing(dnssecsd.org)
------------------------------


# Information
name servers
	a.mail.sd
	b.mail.sd


Install random number generator
===============================

note: it is unpredictable random number generator to enhance the encryption operation in the server

install
$ sudo apt install haveged

	start the service
$ sudo systemctl enable haveged
$ sudo systemctl start haveged

	veryfiy
$ ps -aux | grep haveged


Signiner server configuration
=============================

1. Enable DNSSEC in the configuration file


	Check the bind9 version
		# named -V
	
	Create keys directory
		# mkdir /var/bind9/chroot/var/cache/bind/keys
	
	Configure options
		# vim /etc/bind/named.conf.options

	options {
    ...
    key-directory "/var/cache/bind/keys";
	dnssec-enable yes;
    ...
	};

2. Generate key pairs (KSK, ZSK)

Algorithm 13, Elliptic Curve Digital Signing Algorithm (ECDSA) was selected.

	Generate ZSK
		# cd /var/bind9/chroot/var/cache/bind/keys
		# dnssec-keygen -3 -a ECDSAP256SHA256 dnssecsd.org
	
	Generate KSK
		# dnssec-keygen -fk -3 -a ECDSAP256SHA256 dnssecsd.org
		# chown -R bind:root /var/bind9/chroot/var/cache/bind/keys

info:
	-3	NSEC3	; https://datatracker.ietf.org/doc/rfc9276/
	-a	Algorithm
	-fk	flag ksk ; to be reviewed and replaced by 'KSK'

	

note: Algorithm 13 is selected 
source: https://www.iana.org/assignments/dns-sec-alg-numbers/dns-sec-alg-numbers.xhtml
		https://www.rfc-editor.org/rfc/rfc6605.html

3. Publish the public key

	Ensure the generated keys are located in the directory path included in the named.conf.option file.
	fix the chown to bind:root

4. Signing the .sd zone and Publish the new zone file

	inline Signing will be automated by bind9. add the following configuration in the .sd zone part.

		# vim /etc/bind/named.conf.local


	.....
zone "dnssecsd.org." {
        type master;
        file "master/db.dnssecsd.org";
        masterfile-format text;
        # publish and activate dnssec keys:
		auto-dnssec maintain;
		# use inline signing:
        inline-signing yes;
};
	.....

	use rndc to load the new keys and do inline signing
		# rndc reload
		# rndc reconfig
		# rndc loadkeys dnssecsd.org

note: the generated files will not be readable, the file will be like "db.sd.signed", the zone configuration will point to the new signed file.

5. Testing

		dig @localhost +dnssec +multiline dnssecsd.org any
		dig @localhost +dnssec dnssecsd.org any
		http://dnsviz.net
		https://dnssec-analyzer.verisignlabs.com/


6. Push the DS record to the parent (.org)
	RFC 7344	automating DNSSEC delegation trust maintenance
	RFC 7477	child to parent synchronization in dns


		# dnssec-dsfromkey <KSK.key public key>


Note: The DS record should be added to the parent zone using the registrar/registry panel.


7. Testing again once the DS is in the parent zone. 

		dig @localhost +dnssec +multiline dnssecsd.org any
		dig @localhost +dnssec dnssecsd.org any
		http://dnsviz.net
		https://dnssec-analyzer.verisignlabs.com/

	
