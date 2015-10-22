# Tutoriel 2 : Entrerprise Java Beans





## Mise en œuvre d'un couche de service sans état 

Nous allons mettre en œuvre une couche métier pour gérer les livres de notre *bookstore*. Pour ce faire, nous nous appuyons sur un *Stateless Session Bean*, c'est à dire un *Session Bean* qui ne garde pas l'état d'un client spécifique.  


### Service local

#### Interface

1. Créez un package `fr.emn.gsi2015.bookstore.business` pour stocker les classes *Beans* de la couche métier.
2. Créez une interface `BookLocalService` définissant les méthodes `List<Book> findBooks()`, `Book findBookById(Long id)` et `Book createBook(Book book)`. 
3. Nous voulons accéder à ces méthodes localement, donc précisez le avec l'annotation `@Local`. 

```java
package fr.emn.gsi2015.bookstore.business;

import java.util.List;

import javax.ejb.Local;

import fr.emn.gsi2015.bookstore.persistence.Book;

@Local
public interface BookLocalService {
        
   List<Book> findBooks();

   Book findBookById(Long id);
   
   Book createBook(Book book);
}

```

#### Session Bean


Maintenant nous pouvons procéder avec la mise en œuvre de notre *session bean*. 

1. Créez une classe *Bean* `BookEJB` dans le package `fr.emn.gsi2015.bookstore.business`. 
2. Nous n'avons pas besoin de garder l'étant d'un client spécifique. De ce fait, nous pouvons définir notre *bean* comme étant *Stateless* avec l'annotation `@Stateless`. 
3. Définissez une variable d'instance du type `EntityManager` permettant manipuler l'entité `Book` sur la base de données. Une astuce pour éviter la gestion manuelle d'ouverture et fermeture de conection 
4. Implémentez les méthodes définies dans l'interface `BookLocalService`. 

Votre code devrait ressembler au code ci-dessous.

*** OBS : *** afin de faciliter la gestion de dépendances, c-a-d la création, destruction d'objets dans un autre objet (ex. : l'objet du type `EntityManager` défini dans `BookEJB`) nous pouvons nous appuyer sur l'injection de dépendences. Il suffit d'utiliser une simple annotation (ex. : `@PersistenceContext(unitName = "bookstore-hsqldb")`) et l'engine d'injection de dépendances s'occupe de tout (oui, c'est magique!).  


```java
public class BookEJB implements BookLocalService {

    @PersistenceContext(unitName = "bookstore-hsqldb")
    private EntityManager em;

    public List<Book> findBooks() {
         TypedQuery<Book> query = em.createNamedQuery("findAllBooks", Book.class);
         return query.getResultList();
    }
    
    
    public Book findBookById(Long id) {
         return em.find(Book.class, id);
    }

    public Book createBook(Book book) {
         em.persist(book);
         return book;
    }
}
```

#### Exécuter

Pour pouvoir utiliser le *Session Bean* `BookEJB` nous avons besoin de créer un conteneur EJB, le charger dans le conteneur. 

1. Créez une classe principal `MainLocal` dans le package `fr.emn.gsi2015.bookstore.business`.
2. Définissez des propriétés du conteneur permettant de charger les EJB à partir des `*EJB.class` dans le répertoire `target/classes` de votre projet.   
 
```java
Map<String, Object> properties = new HashMap<>();
properties.put(EJBContainer.MODULES, new File("target/classes"));
```

3. Créez une instance du containeur et récuperez son contexte. 

```java
try (EJBContainer ec = EJBContainer.createEJBContainer(properties)) {
    Context ctx = ec.getContext();
    
    ...

} catch (NamingException e) {
    e.printStackTrace();
}
```

4. Cherchez le EJB via JNDI en suivant le standard `java:<scope>[/<app-name>]/<module-name>/<bean-name>[!<fully-qualified-interface-name>]`.

```java
BookLocalService bookEJB = (BookLocalService) ctx
     .lookup("java:global/classes/BookEJB!fr.emn.gsi2015.bookstore.business.BookLocalService");
``` 
5. Manipulez le *Session Bean* pour lister, ajouter et afficher les livres existants dans la base de données. 

Voici à quoi votre classe principal doit ressembler.

```java
package fr.emn.gsi2015.bookstore.business;


import java.io.File;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.ejb.embeddable.EJBContainer;
import javax.naming.Context;
import javax.naming.NamingException;

import fr.emn.gsi2015.bookstore.persistence.Book;

public class MainLocal {
   public static void main(String[] args) {
       Map<String, Object> properties = new HashMap<>();
       properties.put(EJBContainer.MODULES, new File("target/classes"));

       try (EJBContainer ec = EJBContainer.createEJBContainer(properties)) {
           Context ctx = ec.getContext();

           // Loading session bean
           BookLocalService bookEJB = (BookLocalService) ctx
                           .lookup("java:global/classes/BookEJB!fr.emn.gsi2015.bookstore.business.BookLocalService");

           System.out.println("========== LISTING ALL AVAILABLE BOOKS ============");

           // Lists all books
           List<Book> books = bookEJB.findBooks();
           for (Book b : books) {
                   System.out.println(b);
           }

           System.out.println("========== CREATING A NEW BOOK : H2G2 ============");

           // Creates a new book and Persists the book to the database
           Book book = new Book("H2G2", "The Hitchhiker's Guide to the Galaxy", 12.5F, "1-84023-742-2", 354, false);
           bookEJB.createBook(book);

           System.out.println("========== RETRIEVING BOOK WHOSE ID IS 1001L ============");

           // Searches for a book whose id is 1001
           Book book2 = bookEJB.findBookById(1001L);
           System.out.println(book2);

           
           
           
           ctx.close();
           ec.close();
    
       } catch (NamingException e) {
               e.printStackTrace();
       }
   }
}
```

Pour exécuter cette classe, il suffit de cliquer droit sur la classe > Run As > Java Application. 

La sortie doit ressember à ceci.

```
...
========== LISTING ALL AVAILABLE BOOKS ============
Book [id=1000, title=Beginning Java EE 6, price=49.0, description=Best Java EE book ever, isbn=1234-5678, nbOfPage=450, illustrations=true]
Book [id=1001, title=Beginning Java EE 7, price=53.0, description=No, this is the best , isbn=5678-9012, nbOfPage=550, illustrations=true]
Book [id=1010, title=The Lord of the Rings, price=23.0, description=One ring to rule them all, isbn=9012-3456, nbOfPage=222, illustrations=false]
========== CREATING A NEW BOOK : H2G2 ============
========== RETRIEVING BOOK WHOSE ID IS 1001L ============
Book [id=1001, title=Beginning Java EE 7, price=53.0, description=No, this is the best , isbn=5678-9012, nbOfPage=550, illustrations=true]
...

```

### Service distant (*Remote Service*)  

Parfois il est impérative de faire communiquer des objets qui logiquement (ou
même physiquement) distant, c'est à dire, des objets qui sont deployés sur
différents conteneurs (différents Java VM), différents serveurs, machines
physique, etc. 

Pour rendre accessible à distance un *Session Bean*, il suffit de l'annoter avec `@Remote`. 


#### Interface

1. Créez une interface `BookRemoteService` dans le package `fr.emn.gsi2015.bookstore.business`
2. Définissez la méthode `List<Book> findBooksRemotely()`
3. Précisez, avec l'annotation `@Remote`, qu'il s'agit bien d'un service distant. 

```java
package fr.emn.gsi2015.bookstore.business;

import java.util.List;
import javax.ejb.Remote;
import fr.emn.gsi2015.bookstore.persistence.Book;

@Remote
public interface BookRemoteService  {        
   List<Book> findBooksRemotely();
}
```

#### Session Bean

Modifiez la classe `BookEJB` pour qu'elle implémente aussi l'interface `BookRemoteService`.

```java
@Stateless
public class BookEJB implements BookLocalService , BookRemoteService  {
        ...

    //accessed remotely via remote method interface
    public List<Book> findBooksRemotely() {
        return this.findBooks();
    }        
        
}
```

#### Entités *Serializable* 

Pour que les services soient accessible via le réseau, tous les classes entités (dans ce cas la classe `Book`) doivent implémenter l'interface `Serializable`. Ainsi, modifiez la classe `Book`, comme indiqué ci-dessous. 

```java
public class Book implements Serializable {
   ...
}

```

#### Serveur d'application

Cette fois-ci nous allons utiliser le serveur d'application *Glassfish* pour
pouvoir accéder à notre EJB.  Pour cela, nous allons nous appuyer sur le plugin
Maven `maven-embedded-glassfish-plugin`. Il suffit de tapez `mvn clean package
embedded-glassfish:run`, depuis la racine du projet, pour packager notre projet, lancer le serveur
*Glassfish* tout en deployant le package généré contenant les classes entités
et EJBs.     

Sur Eclipse, il faut proceder comme suit :

1. Clique droit sur le projet > Run As  > Maven Build...
2. Donnez un nom à votre configuration (ex. : bookstore glassfish server) 
3. Entrez les *Maven Goals* : `clean package embedded-glassfish:run`.
4. Appuyez sur `Apply` et ensuite `Run`.
![alt text](./glassfish.png)


Ça y est! Votre serveur projet sera packagé et deployé sur le serveur d'application *Glassfish*. 

```
...
Oct 22, 2015 6:12:50 PM com.sun.enterprise.web.WebApplication start
INFO: Loading application [bookstore-0.0.1-SNAPSHOT] at [/bookstore]
Oct 22, 2015 6:12:50 PM org.glassfish.deployment.admin.DeployCommand execute
INFO: bookstore-0.0.1-SNAPSHOT was successfully deployed in 11,552 milliseconds.
Oct 22, 2015 6:12:50 PM PluginUtil doDeploy
INFO: Deployed bookstore-0.0.1-SNAPSHOT
Hit ENTER to redeploy, X to exit
```

#### Client distant

Pour pouvoir utiliser le *Session Bean* distant nous allons tout d'abord
définir un client (une classe principale jouant le rôle d'un client).

1. Créez une classe principale `MainRemote` dans le package `fr.emn.gsi2015.bookstore.business`
2. Dans la méthode main créez un contexte initial avec des propriétés permettant l'accès à des EJB distants via JNDI. 
3. Récupérer le *Session Bean* `BookEJB` à partir de l'interface `BookRemoteService` via JNDI. 
4. Appelez la méthode `findBooksRemotely` et affichez le résultat. 

Voici à quoi doit ressembler la classe principale `MainRemote` : 

```java
package fr.emn.gsi2015.bookstore.business;

import java.util.Properties;
import javax.naming.InitialContext;
import javax.naming.NamingException;

public class MainRemote {

    public static void main(String[] args) {
       try {
           Properties props = new Properties();
           String host = "localhost";
           String port = "3700";
      
    	   props.setProperty("java.naming.factory.initial",
           "com.sun.enterprise.naming.SerialInitContextFactory");
           props.setProperty("java.naming.factory.url.pkgs",
                "com.sun.enterprise.naming");
           props.setProperty("java.naming.factory.state",
                "com.sun.corba.ee.impl.presentation.rmi.JNDIStateFactoryImpl");
           props.setProperty("org.omg.CORBA.ORBInitialHost", host);
           props.setProperty("org.omg.CORBA.ORBInitialPort", port);

           InitialContext ic = new InitialContext(props);
        
           System.out.println("=========== Looking for remote session bean =============");
           BookRemoteService bookEJB = (BookRemoteService) ic.lookup("fr.emn.gsi2015.bookstore.business.BookRemoteService");
           System.out.println("=========== Remote session bean found =============="); 
           System.out.println("=========== Retrieving books now! ============");
           System.out.println(bookEJB.findBooksRemotely());


       } catch (NamingException ex) {
           e.printStackTrace();
       }

    }

}
```

Pour exécuter le client il suffit de cliquer droit sur la classe `MainRemote` > Run As > Java Application. 

La sortie de votre programme doit ressemble à ça :

```
=========== Looking for remote session bean =============
=========== Remote session bean found ==============
=========== Retrieving books now! ============
[Book [id=1000, title=Beginning Java EE 6, price=49.0, description=Best Java EE book ever, isbn=1234-5678, nbOfPage=450, illustrations=true], Book [id=1001, title=Beginning Java EE 7, price=53.0, description=No, this is the best , isbn=5678-9012, nbOfPage=550, illustrations=true], Book [id=1010, title=The Lord of the Rings, price=23.0, description=One ring to rule them all, isbn=9012-3456, nbOfPage=222, illustrations=false]]
```






 
 


## Mise en œuvre d'un panier avec un *Stateful EJB* 

