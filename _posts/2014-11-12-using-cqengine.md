---
layout: post
title: Using CQengine
modified: 2014-11-12
excerpt: A simple Example on Querying collections using Cqengine in Java
tags: [Cqengine , code, java]
date: 2014-11-12T18:20:43+05:30
---

Cqengine in short is a in-memory DataBase. It is a NoSQL indexing and Query Engine, for retrieving objects matching SQL-like queries from Java collections, with ultra-low latency. Supports millions of queries per second, with response times in the order of microseconds.

#### Maven Dependency

{% highlight xml %}
<dependency>
	<groupId>com.googlecode.cqengine</groupId>
	<artifactId>cqengine</artifactId>
	<version>1.2.7</version>
</dependency>
{% endhighlight %}

#### SBT
{% raw %}
libraryDependencies += "com.googlecode.cqengine" % "cqengine" % "1.2.7"
{% endraw %}

### Adding Attributes

First Step is to create a class with the revelant Objects and add Attributes which can be indexed and queried with.

######The initial Class Structure without the attributes
{% highlight java %}
import java.util.List;
public class User {
    private String userName;
    private String email;
    private String firstName;
    private String lastName;
    private String profession;

    public User(String userName, String email, String firstName, String lastName, String profession) {
        this.userName = userName;
        this.email = email;
        this.firstName = firstName;
        this.lastName = lastName;
        this.profession = profession;
    }
}
{% endhighlight %}

#### Attributes
In the Below code we are adding a SimpleAttribute for the UserName Values so that the User Name can be indexed and queried with.
{% highlight java %}
public static final Attribute<User , String> USER_NAME =  new SimpleAttribute<User, String>() {
    @Override
    public String getValue(User user) {
        return user.getUserName();
    }
};
{% endhighlight %}


###### The Complete Class with attributes for email username and profession
{% highlight java %}
import com.googlecode.cqengine.attribute.Attribute;
import com.googlecode.cqengine.attribute.SimpleAttribute;

public class User {
    private String userName;
    private String email;
    private String firstName;
    private String lastName;
    private String profession;

    public User(String userName, String email, String firstName, String lastName, String profession) {
        this.userName = userName;
        this.email = email;
        this.firstName = firstName;
        this.lastName = lastName;
        this.profession = profession;
    }

    public static final Attribute<User, String> USER_NAME = new SimpleAttribute<User, String>() {
        @Override
        public String getValue(User user) {
            return user.getUserName();
        }
    };
    public static final Attribute<User, String> PROFESSION = new SimpleAttribute<User, String>() {
        @Override
        public String getValue(User user) {
            return user.getProfession();
        }
    };
    public static final Attribute<User, String> EMAIL = new SimpleAttribute<User, String>() {
        @Override
        public String getValue(User user) {
            return user.getEmail();
        }
    };
    public static final Attribute<User, String> FISRT_NAME = new SimpleAttribute<User, String>() {
        @Override
        public String getValue(User user) {
            return user.getFirstName();
        }
    };
    public String getUserName() {
        return userName;
    }

    public String getEmail() {
        return email;
    }

    public String getFirstName() {
        return firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public String getProfession() {
        return profession;
    }

}

{% endhighlight %}

#### Preparing the Index 

Lets create a Indexed Collection that we can use for the User Objects.
{% highlight java %}
    private static IndexedCollection indexedUsers = CQEngine.newInstance();
{% endhighlight %}

Adding an attribute to the IndexedCollection so that they will be indexed.
{% highlight java %}
    indexedUsers.addIndex(HashIndex.onAttribute(User.USER_NAME));
{% endhighlight %}

Adding User to the indexed Collection
{% highlight java %}
    public static void addUser(User user) {
        indexedUsers.add(user);
    }
{% endhighlight %}



{% highlight java %}
import com.googlecode.cqengine.CQEngine;
import com.googlecode.cqengine.IndexedCollection;
import com.googlecode.cqengine.index.hash.HashIndex;
public class UserCollections {
    private static IndexedCollection indexedUsers = CQEngine.newInstance();
    public static void initialize() {
        indexedUsers.addIndex(HashIndex.onAttribute(User.USER_NAME));
        indexedUsers.addIndex(HashIndex.onAttribute(User.EMAIL));
        indexedUsers.addIndex(HashIndex.onAttribute(User.PROFESSION));
        indexedUsers.addIndex(HashIndex.onAttribute(User.FISRT_NAME));
    }
    public static void addUser(User user) {
        indexedUsers.add(user);
    }
}
{% endhighlight %}

#### Querying

Lets do a simple Query which would return the users having a given profession
{% highlight java %}
public static List<User> getUsersWithProfession(String profession){
    List<User> users = new ArrayList<User>();
    for(Object obj :indexedUsers.retrieve(equal(User.PROFESSION,profession))){
        users.add((User) obj);
    }
    return users;
}
{% endhighlight %}

A Query for searching users with a given profession and a spefic First name
{% highlight java %}
public static List<User> getUsersWithProfessionAndName(String profession,String firstName){
    List<User> users = new ArrayList<User>();
    for(Object obj :indexedUsers.retrieve(and(equal(User.PROFESSION,profession),equal(User.FIRST_NAME,firstName)))){
        users.add((User) obj);
    }
    return users;
}
{% endhighlight %}

Lets Say you have multiple Queries its a good practice to use a list of queries instead of just a query
{% highlight java %}
public static List<User> getUsersWithProfession(List<String> professions) {
    List<User> users = new ArrayList<User>();
    List<Query<User>> userQueries = new ArrayList<Query<User>>();
    for (String profession : professions) {
        userQueries.add(equal(User.PROFESSION, profession));
    }
    Query<User> finalQuery;
    if (userQueries.size() == 1) {
        finalQuery = userQueries.get(0);
    } else {
        finalQuery = or(userQueries);
    }
    for (Object obj : indexedUsers.retrieve(finalQuery)) {
        users.add((User) obj);
    }
    return users;
}
{% endhighlight %}



###### Complete Class UserCollections
{% highlight java %}
package in.cybergen.blog;

import com.googlecode.cqengine.CQEngine;
import com.googlecode.cqengine.IndexedCollection;
import com.googlecode.cqengine.index.hash.HashIndex;
import com.googlecode.cqengine.query.Query;

import java.util.ArrayList;
import java.util.List;

import static com.googlecode.cqengine.query.QueryFactory.and;
import static com.googlecode.cqengine.query.QueryFactory.equal;
import static com.googlecode.cqengine.query.QueryFactory.or;

/**
 * Created by vishnu on 12/11/14.
 */
public class UserCollections {
    private static IndexedCollection indexedUsers = CQEngine.newInstance();

    public static void initialize() {
        indexedUsers.addIndex(HashIndex.onAttribute(User.USER_NAME));
        indexedUsers.addIndex(HashIndex.onAttribute(User.EMAIL));
        indexedUsers.addIndex(HashIndex.onAttribute(User.PROFESSION));
    }

    public static void addUser(User user) {
        indexedUsers.add(user);
    }

    public static List<User> getUsersWithProfessionAndName(String profession, String firstName) {
        List<User> users = new ArrayList<User>();
        for (Object obj : indexedUsers.retrieve(and(equal(User.PROFESSION, profession), equal(User.FIRST_NAME, firstName)))) {
            users.add((User) obj);
        }
        return users;
    }

    public static List<User> getUsersWithProfession(String profession) {
        List<User> users = new ArrayList<User>();
        for (Object obj : indexedUsers.retrieve(equal(User.PROFESSION, profession))) {
            users.add((User) obj);
        }
        return users;
    }

    public static List<User> getUsersWithProfession(List<String> professions) {
        List<User> users = new ArrayList<User>();
        List<Query<User>> userQueries = new ArrayList<Query<User>>();
        for (String profession : professions) {
            userQueries.add(equal(User.PROFESSION, profession));
        }
        Query<User> finalQuery;
        if (userQueries.size() == 1) {
            finalQuery = userQueries.get(0);
        } else {
            finalQuery = or(userQueries);
        }
        for (Object obj : indexedUsers.retrieve(finalQuery)) {
            users.add((User) obj);
        }
        return users;
    }

    public static void main(String[] args) {
        initialize();
        addUser(new User("vishnu667", "vishnu@i.com", "vishnu", "prasad", "programmer"));
        addUser(new User("sam675", "sam@i.com", "sam", "jenrich", "lawyer"));
        addUser(new User("john321", "john@i.com", "john", "paul", "manager"));
        addUser(new User("nirmal047", "nimmmy@i.com", "nirmal", "kumar", "programmer"));
        System.out.println("Users with Profession ");
        listUsers(getUsersWithProfession("programmer"));
        System.out.println("Users with Profession and first name");
        listUsers(getUsersWithProfessionAndName("programmer", "nirmal"));
        System.out.println("Users with the given Professions ");
        List<String> professions = new ArrayList<String>();
        professions.add("programmer");
        professions.add("lawyer");
        listUsers(getUsersWithProfession(professions));
    }

    public static void listUsers(List<User> users) {
        for (User user : users) {
            System.out.println("UserName " + user.getUserName() + " email " + user.getEmail() + " firstName " + user.getFirstName() + " Profession " + user.getProfession());
        }
    }
}

{% endhighlight %}

{% highlight shell-session%}
Users with Profession 
UserName nirmal047 email nimmmy@i.com firstName nirmal Profession programmer
UserName vishnu667 email vishnu@i.com firstName vishnu Profession programmer
Users with Profession and first name
UserName nirmal047 email nimmmy@i.com firstName nirmal Profession programmer
Users with the given Professions 
UserName nirmal047 email nimmmy@i.com firstName nirmal Profession programmer
UserName vishnu667 email vishnu@i.com firstName vishnu Profession programmer
UserName sam675 email sam@i.com firstName sam Profession lawyer
{% endhighlight %}