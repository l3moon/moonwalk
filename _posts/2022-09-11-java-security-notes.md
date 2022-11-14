---
layout: post
author: L3moon龙
tags: [java, JNDI Injection]
---
# Analysis of JNDI injection in Java 
## 0x01 Basic Concept 
### What is JNDI 
- The full name of JNDI is Java Naming and DirectoryInterface (Java Naming and Directory Interface), which is a set of application program interfaces that provide developers with a unified general interface for finding and accessing various resources, which can be used to define users, networks, machines, objects and services and other resources.

The services supported by JNDI mainly include: DNS, LDAP, CORBA, RMI, etc.
Simply put, JNDI is a set of API interfaces. Each object has a unique set of key-value bindings that bind the name to the object. The specified object can be retrieved by name, and the object may be stored in RMI, LDAP, CORBA, etc.
As shown in the figure: 

![image](https://cdn.educba.com/academy/wp-content/uploads/2019/11/Architecture.png)

#### Java Naming
A naming service is a binding of key-value pairs that enables applications to retrieve values ​​by key.
#### Java Directory
Directory services are a natural extension of naming services. The difference between the two is that objects in a directory service can have attributes, while objects in a naming service have no attributes. Therefore, objects can be searched based on attributes in the directory service.

- JNDI allows you to access files in the file system, locate remote RMI-registered objects, access directory services such as LDAP, and locate EJB components on the network.
ObjectFactory
#### Object Factory 
is used to convert data stored in Naming Service (such as RMI/LDAP) into expressible data in Java, such as objects in Java or basic data types in Java. Each Service Provider may be equipped with multiple Object Factory.
The problem with JNDI injection is in the custom ObjectFactory class that can be downloaded remotely.

#### JNDI Code Example 
Binding and lookup methods are provided in JNDI:
    -bind: bind the name to the object;
    -lookup: object executed by name retrieval;

let's first write a Secret class with secret key and secret value properties:

```java
public class Secret Implements Remote, Serializable {
    private static final long serialVersionUID = 1L; 
    private String key;
    private String value;

    public void setKey(String key) {
        this.key = key;
    }

    public void setValue(String value) {
        this.value = value;
    }

    public String getKey() {
        return key;
    }

    public String getValue() {
        return value;
    }

    public String toString() {
        return "key: " + key + " value: " + value;
    } 
}
```
server.java actually represents the server side and the client side together.
the first part is the initSecret() method which the srver that implements the RMI service throught the JNDI and binds the instantiated secret object to the RMI service through the JNDI bind() method.
the second part is the client part which will be represented via the findsecret() method which will use the JNDI lookup() method to find the secret object and print the secret object.

```java
public class Server {
    public static void main(String[] args) {
        try {
            initSecret();
            findSecret();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void initSecret() throws Exception {
        // create a secret object
        Secret secret = new Secret();
        secret.setKey("secretKey");
        secret.setValue("secretValue");

        // create a JNDI initial context
        Hashtable<String, String> env = new Hashtable<String, String>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        Context ctx = new InitialContext(env);

        // bind the secret object to the RMI service
        ctx.bind("rmi://localhost:1099/SecretObj", secret);
    }

    public static void findSecret() throws Exception {
        // create a JNDI initial context
        Hashtable<String, String> env = new Hashtable<String, String>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        Context ctx = new InitialContext(env);

        // lookup the secret object
        Secret secret = (Secret) ctx.lookup("rmi://localhost:1099/SecretObj");
        System.out.println(secret);
    }
}
```

That's not the only possible implementation of JNDI, there are other implementations such as LDAP, CORBA, etc. and the implementation of JNDI is not limited to RMI, it can also be implemented through other protocols such as HTTP, etc. 

this an example of jndi implementation through http protocol: 

 ```java
public class Server {
    public static void main(String[] args) {
        try {
            initSecret();
            findSecret();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void initSecret() throws Exception {
        // create a secret object
        Secret secret = new Secret();
        secret.setKey("secretKey");
        secret.setValue("secretValue");

        // create a JNDI initial context
        Hashtable<String, String> env = new Hashtable<String, String>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.url.http.HttpURLContextFactory");
        env.put(Context.PROVIDER_URL, "http://localhost:8080");
        Context ctx = new InitialContext(env);

        // bind the secret object to the RMI service
        ctx.bind("http://localhost:8080/SecretObj", secret);
    }

    public static void findSecret() throws Exception {
        // create a JNDI initial context
        Hashtable<String, String> env = new Hashtable<String, String>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.url.http.HttpURLContextFactory");
        env.put(Context.PROVIDER_URL, "http://localhost:8080");
        Context ctx = new InitialContext(env);

        // lookup the secret object
        Secret secret = (Secret) ctx.lookup("http://localhost:8080/SecretObj");
        System.out.println(secret);
    }

} 

```
Note : 
    Server: when initializing and configuring JNDI settings, you need to pre-specify its context environment, such as specifying it as an RMI service, and finally call javax.naming.InitialContext.bind() to bind the specified object. to the RMI registry;

#### Reference class

- The Reference class represents a reference to an object that exists outside the naming/directory system.

In order to store Object objects in the Naming or Directory service, Java provides the Naming Reference function. Objects can be stored in the Naming or Directory service by binding the Reference, such as RMI, LDAP, etc.
When using Reference, we can directly write the object in the constructor, and when it is called, the method of the object will be triggered.
Several key attributes: 

    className: The class name used for remote loading; 

    classFactory: The name of the class to be instantiated in the loaded class;

    classFactoryLocation: The address of the remote loaded class, the address for providing classes data can be a protocol such as file/ftp/http;

#### Java Security Manager 

- Objects in Java are divided into local objects and remote objects. Local objects are trusted by default, but remote objects are not trusted. For example, when our system loads an object from a remote server, for the sake of security, the JVM will limit the ability of the object, such as prohibiting the object from accessing our local file system, etc. These rely on the security manager in the existing JVM (SecurityManager) to achieve. 

![image](https://docs.oracle.com/en/java/javase/12/security/img/jssec_dt_030_anc4.png)

#### Security Manager for JNDI  
This part will also be discussed in detail later in bypassing the restrictions of higher versions of JDK. i need you to concentrate 

so to load remote objects, JDNI has two different security control methods. For Naming Manager, the rules of the relative security manager are relatively broad, but the JNDI SPI layer will be controlled according to the rules in the following table: 
![image](https://img-blog.csdn.net/20160825165548350?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### JNDI protocol dynamic conversion 
in the RMI service implemented by JNDI, the context environment (RMI, LDAP, CORBA, etc.) can be pre-specified when initializing and configuring the JNDI settings. Here are the two previous writing methods: 
```java
    Hashtable<String, String> env = new Has htable<String, String>();
    env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
    env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
    Context ctx = new InitialContext(env);
    ```
```java
    Hashtable<String, String> env = new Hashtable<String, String>();
    env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.url.http.HttpURLContextFactory");
    env.put(Context.PROVIDER_URL, "http://localhost:8080");
    Context ctx = new InitialContext(env);
```
However, when calling lookup() or search(), you can use the dynamic conversion context with URI. For example, if the current context has been set above to access the RMI service, you can directly use the LDAP URI format to convert the context to access the LDAP service. The binding object instead of the original RMI service:
```java
    // lookup the secret object
    Secret secret = (Secret) ctx.lookup("ldap://localhost:389/SecretObj");
    System.out.println(secret);
```

and this is what's going on in the background:
```java
    protected  Context  getURLOrDefaultInitCtx (Name paramName)  throws  NamingException  { 
    if  (NamingManager.hasInitialContextFactoryBuilder()) { 
        return  getDefaultInitCtx();  
    } 
    if  (paramName.size() >  0 ) { 
        String str1 = paramName.get( 0 ); 
        String str2 = getURLScheme(str1);  // try to parse the protocol in the URI 
        if  (str2 !=  null ) { 
            // If there is a Schema protocol, try to get its corresponding context 
            Context localContext = NamingManager.getURLContext(str2,  this .myProps); 
            if  (localContext !=  null ) {  
                return  localContext; 
            } 
        }   
    } 
    return  getDefaultInitCtx(); 
} 
```
The above code is the implementation of the getURLOrDefaultInitCtx() method in the InitialContext class. When the lookup() method is called, the getURLOrDefaultInitCtx() method is called to get the context. If the URI contains a protocol, the corresponding context will be obtained. If not, the default context will be obtained.

> in the next days i'll complete the rest of the article and how to abuse jndi injection TwT 
made with love by l3mon <3




