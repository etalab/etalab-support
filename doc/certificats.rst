***********************
Gestion des certificats
***********************


On utilise `StartSSL <https://www.startssl.com/>`_ comme fournisseur de certificats.


Création d'un certificat OpenSSL/TLS
====================================


Création d'une clé privée
-------------------------

::

  mkdir data.gouv.fr-certificates
  cd data.gouv.fr-certificates
  certtool --generate-privkey --outfile wildcard.data.gouv.fr-private-key-raw.pem
    Generating a 2432 bit RSA private key...


Création d'une Certificate Signing Request
------------------------------------------

::

 certtool --generate-request --load-privkey wildcard.data.gouv.fr-private-key-raw.pem --outfile wildcard.data.gouv.fr-csr.pem
    Generating a PKCS #10 certificate request...
    Common name: *.data.gouv.fr
    Organizational unit name: 
    Organization name: 
    Locality name: 
    State or province name: 
    Country name (2 chars): FR
    Enter the subject's domain component (DC): 
    UID: 
    Enter a dnsName of the subject of the certificate: *.data.gouv.fr
    Enter a dnsName of the subject of the certificate: 
    Enter a URI of the subject of the certificate: 
    Enter the IP address of the subject of the certificate: 
    Enter the e-mail of the subject of the certificate: 
    Enter a challenge password: 
    Does the certificate belong to an authority? (y/N): 
    Will the certificate be used for signing (DHE and RSA-EXPORT ciphersuites)? (Y/n): n
    Will the certificate be used for encryption (RSA ciphersuites)? (Y/n): n
    Is this a TLS web client certificate? (y/N): n
    Is this a TLS web server certificate? (y/N): y

.. note:: En fait il semble qu'aucun champ de la CSR ne soit utilisé par StartTLS.

