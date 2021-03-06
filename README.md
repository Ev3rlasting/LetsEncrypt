## Creating a java keystore for tomcat using letsencrypt

### Run letsencrypt

Something like this:
./letsencrypt-auto certonly --standalone --email stu26code@gmail.com -d www.shareplaylearn.com -d shareplaylearn.com -d www.drunkscifi.com -d drunkscifi.com -d www.shareplaylearn.net -d shareplaylearn.net

### Stop, and reconsider nginx

So, you want to run tomcat with ssl do you? How about running tomcat in the clear, and fronting it with nginx, which doesn't require you to deal with the java keystore. You don't want to do that? You sure? The java keystore is an abomination, you know that? You like tomcat? OK. Keep in mind, though, that you and I are both wrong, and we should really move off running ssl in our tomcat. I already have nginx fronting it anyways....

## Create a java keystore
It should be understood that the java keytool program used create keystores is a complete piece of junk. It will give you errors like "NullPointerException, invalid input" if you don't give it a pkcs12 archive with a password.
Furthermore, it doesn't understand the PEM format that any sane program dealing with certificates would. In short, take 10 deep breathes before using it, and don't give up - it's probably the keytool doing something very, very, very stupid.

 - take your nice pem files from letsencrypt, and create a ugly, pkcs12 formatted file with your private and public entries:
     ```
      $>sudo openssl pkcs12 -export -in cert.pem -inkey privkey.pem -out [pkcs_filename].p12 -name [name]
     ```
   Keep in mind that you'll be asked to add a password to this temp file. This may seem stupid because you have the private key sitting in plaintext in the privkey.pem file right next to your output.
   However, remember - keytool is also stupid, and will NPE if you use an empty password.

 - take your pkcs12 file, and create a java keystore from it:
    ```
      $> sudo keytool -importkeystore -destkeystore [jks_filename].jks -srckeystore [pkcs_filename].p12 -srcstoretype PKCS12 -alias [name]
    ```
   It will then ask for your export password - this is what it will encrypt the jks with. Make this a good one!
   Then it will ask for the password used in the pkcs12 file above. Yes, keytool is stupid and asks for the import password after the export one, and fails instantly, right at the end, if something goes wrong here.

 - [optional, but likely] Hopefully, the only reason you got stuck doing this is your running some webapp in tomcat. Now configure your tomcat like so:
    
      ```
        <Connector
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           port="8443" maxThreads="200"
           scheme="https" secure="true" SSLEnabled="true"
           keystoreFile="${user.home}/.keystore" keystorePass="changeit"
           clientAuth="false" sslProtocol="TLS"/>
      ```

 - And finally, you will probably want to add the cert chain for your shiny new cert into your jvm cacerts, so java will trust your privkey. You need this if you app needs to trust itself - like if you have callback.
   If you haven't changed your cacert password (likely), it is "changeit"
   ```
    keytool --importcert -file fullchain.pem -keystore /usr/lib/jvm/default-java/jre/lib/security/cacerts -v -alias [name]_chain
   ``` 
   And that should do it! 

### Only real reason why
You're using tomcat instead of  <a href="http://sparkjava.com/">Spark Java</a> (Again, why? Maybe legacy??), and you need true end-to-end encryption. For example, you have tomcat on one server in the cloud, and nginx on another, and on some clouds (like digital ocean - hereafter DO), the "private networks" are only private outside the DC. Unless DO says differently, that means anyone with a machine in the same DC can eavesdrop. Of course, your provider will ALWAYS be technically able to eavesdrop, as well as anyone with legal leverage over the provider, but it does prevent random customers on DO from seeing your traffic. But really? you should use sparkjava, co-located with nginx, and bind only to loopback. 
