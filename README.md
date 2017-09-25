# Part 1: MicroProfile Meeting Application

## Overview
This sample application solves a real problem that the Liberty development team had. The globally-distributed Liberty development team has a lot of online meetings using IBM Connections Cloud Meetings. IBM Connections Cloud provides meeting rooms to individual employees, which is a problem for team meetings if the person who initially set up the meeting room can’t make it (e.g. they were called into another meeting, are on vacation, or sick). The sample application provides a single URL for a meeting, which can then be ‘started’ by one person and everyone else gets redirected.

The purpose of this lab is to go over using MicroProfile 1.0, so it will just cover the backend Java logic, and not the user interface, which is provided in GitHub.

Adapted from the blog post: [Writing a simple MicroProfile application](https://developer.ibm.com/wasdev/docs/writing-simple-microprofile-application/)

## Prerequisites
 * [Eclipse Java EE IDE for Web Developers](http://www.eclipse.org/downloads/)
 * IBM Websphere Application Liberty Developer Tools (WDT)
   1. Start Eclipse
   2. Launch the Eclipse Marketplace: **Help** -> **Eclipse Marketplace**
   3. Search for **IBM Websphere Application Liberty Developer Tools**, and click **Install** with the defaults configuration selected
 * [Git](https://git-scm.com/downloads)

## Steps
### Step 1. Check out the source code

  #### From the command line
  Run the following commands:
  
  ```
  $    git clone https://github.com/IBM/microprofile-meeting.git
  ```

  #### In Eclipse, import the project as an existing project.
  1. In Eclipse, switch to the Git perspective.
  2. Click **Clone a Git repository** from the Git Repositories view.
  3. Enter URI `https://github.com/IBM/microprofile-meeting.git`
  4. Click **Next**, then click **Next** again accepting the defaults.
  5. From the **Initial branch** drop-down list, click **master**.
  6. Select **Import all existing Eclipse projects after clone finishes**, then click **Finish**.
  7. Switch to the Java EE perspective.
  8. The meetings project is automatically created in the Project Explorer view.

### Step 2. Creating the MeetingManager class
The first part of this application is a CDI-managed bean that manages the meetings. This bean is application-scoped, meaning there is only one instance. At this point the MicroProfile 1.0 release has not stated a preference for a persistence mechanism, so this example simply stores information in memory. This means that we need shared state, and an application-scoped bean ensures that state is shared by all clients.

1. Create a new class: right-click the `meetings` project, then click **New > Class…**
2. Name the class `MeetingManager`, then click **Finish**.

Eclipse brings up the `MeetingManager` class in the Java editor. The first thing to do is make it a managed bean.

1. Above the class type definition add `@ApplicationScoped`, which is in the package `javax.enterprise.context`:
```java
import javax.enterprise.context.ApplicationScoped;
@ApplicationScoped
public class MeetingManager {
```

2. Save the file.

The next step is to create a map to store the meeting information. There could be concurrent requests so we need to ensure a thread-safe data construct is used:

1. Just after the class definition add the following code:

```java
private ConcurrentMap<String, JsonObject> meetings = new ConcurrentHashMap<>();
```

2. This code introduces three new classes, all of which need to be imported. The relevant imports are:

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import javax.json.JsonObject;
```

3. Save the file

In this example a `JsonObject` is being stored; this is defined by the JSON-Processing standard Java API. Many CRUD-style applications just need to take JSON input and store it away. While you might expect to convert the JSON to some form of object data type, in this application, that would be overkill so we just pass JsonObjects around. Taking this approach somewhat breaks the separation of concerns between business logic and protocol handling so do not take this as a best practice – it’s just a convenient approach for this application.

The bean needs to have four different operations: Add a meeting, get a specific meeting, get all the meetings, and start a meeting. Some will be very simple. To add the operations:

1. Add a method to add a new meeting. This stores the meeting away using the JSON id attribute. It only stores it once, so subsequent calls are essentially ignored.

```java
public void add(JsonObject meeting) {
    meetings.putIfAbsent(meeting.getString("id"), meeting);
}
```

2. Add a method to get a meeting:

```java
public JsonObject get(String id) {
    return meetings.get(id);
}
```

3. Add a method to list all the meetings. This is the second most complicated operation. It essentially loops around all the values in the map created earlier and adds them to a `JsonArrayBuilder` to be returned:

```java
public JsonArray list() {
    JsonArrayBuilder results = Json.createArrayBuilder();
     
    for (JsonObject meeting : meetings.values()) {
        results.add(meeting);
    }
         
    return results.build();
}
```

4. This method introduces three new types: A `JsonArray`, a `JsonArrayBuilder`, and the `Json` class. A `JsonArray` represents an array of `JsonObjects`. A `JsonArrayBuilder` is used to create a `JsonArray`. The `Json` class provides utility methods for constructing the Json builder classes. Add the following imports:

```java
import javax.json.JsonArray;
import javax.json.JsonArrayBuilder;
import javax.json.Json;
```

5. Finally, you need to create the method to start a meeting. The key thing to understand about JsonObjects is that they are read-only. This means that, once they are created, they cannot be changed so, to start a meeting, you need to clone the existing one. This code is hidden in a helper class provided in the Git repository. The method is shown below. It essentially creates a new `JsonObjectBuilder` and copies the entries across. In some cases (not used in this article) the clone may want to not have a field copied across; in that case a list of keys to ignore can be provided.

```java
public static JsonObjectBuilder createJsonFrom(JsonObject user, String ... ignoreKeys) {
    JsonObjectBuilder builder = Json.createObjectBuilder();
    List<String> doNotCopy = Arrays.asList(ignoreKeys);
     
    for (Map.Entry<String, JsonValue> entry : user.entrySet()) {
        if (!!!doNotCopy.contains(entry.getKey())) {
            builder.add(entry.getKey(), entry.getValue());
        }
    }
     
    return builder;
}
```

5. This method introduces two new types: a `JsonValue` and a `JsonObjectBuilder`. A `JsonValue` is the superclass of all the Json types. A `JsonObjectBuilder` is used to create a `JsonObject`. The code also uses the standard Java Collections API. Add the following imports:

```java
import javax.json.JsonValue;
import javax.json.JsonObjectBuilder;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
```

6. When the application user starts a new meeting, their action provides a new `JsonObject` with the meeting ID and a URL for joining the meeting. To ensure the meeting is started, the meeting ID and URL are fetched from the input parameter `meeting`. The `JsonObject` for the existing meeting is fetched from memory as `existingMeeting`. The existing meeting is then cloned using the helper method above and the meeting URL is added by calling add on the `JsonObjectBuilder` returned from the helper method. A `JsonObject` is then built from the builder. Finally, the meetings map has the meeting replaced assuming that the existing meeting is still bound. This ensures thread-safe updates so if two calls to `startMeeting` run at once only one will win.

```java
public void startMeeting(JsonObject meeting) {
    String id = meeting.getString("id");
    String url = meeting.getString("meetingURL");
    JsonObject existingMeeting = meetings.get(id);
    JsonObject updatedMeeting = MeetingsUtil.createJsonFrom(existingMeeting).add("meetingURL", url).build();
    meetings.replace(id, existingMeeting, updatedMeeting);
}
```

7. Save the file.

### Step 3. Creating the MeetingService class
The second part of this example is the JAX-RS service endpoint. This makes a REST API available externally via HTTP.

* Create a new class called `MeetingService`.

Eclipse brings up the `MeetingService` class in the Java editor. Most Java EE beans are automatically considered CDI-managed beans by default, but not JAX-RS beans. JAX-RS beans need to be annotated with a CDI scope to become CDI-managed. In this case we want the normal JAX-RS behaviour of a bean instance per request but we need it to be CDI managed. This can be done using the CDI request scope:

1. Above the class type definition add `@RequestScoped`, which is in the package `javax.enterprise.context`:
```java
import javax.enterprise.context.RequestScoped;
@RequestScoped
 
public class MeetingService {
```

2. Save the file.

The next step is to make the bean a JAX-RS bean. This is done using the JAX-RS Path annotation:

1. Above the class type definition add `@Path`, which is in the package `javax.ws.rs`. This takes a single value which is the default path used to access the JAX-RS resource that this bean will manage:

```java
@Path("meetings")
public class MeetingService {
```

2. This introduces the new type `Path` which needs to be imported:

```java
import javax.ws.rs.Path;
```

3. Save the file.

The meeting service needs two objects injected to perform its actual behaviour. The first is the `MeetingManager` class and the second is a JAX-RS class for managing URI processing:

1. Just after the class definition, add the code below. The `Inject` annotation tells CDI to inject the MeetingManager CDI bean. The `Inject` annotation is from the package `javax.inject`:

```java
@Inject
private MeetingManager manager;
```

2. Import the `Inject` annotation:

```java
import javax.inject.Inject;
```

3. Next, add the code below. The `Context` annotation tells the JAX-RS runtime to inject the `UriInfo` object:

```java
@Context
private UriInfo info;
```

4. The `Context` annotation and `UriInfo` interface are in the `javax.ws.rs.core package` so import the `Context` and `UriInfo`:

```java
import javax.ws.rs.core.Context;
import javax.ws.rs.core.UriInfo;
```

5. Save the file.

The JAX-RS bean needs four methods to respond to the resource requests. JAX-RS uses annotations to work out which methods map to which operations. Although it is common for JAX-RS beans to receive information by annotated content, the JAX-RS specification only allows this when using XML via JAX-B. Every JAX-RS provider supports JSON binding to Java beans so, in general, this isn’t an issue but since this is using MicroProfile we are sticking to JSON-P which is the only required mapping of JSON to Java.

1. The method to add the operation is shown below. The `PUT` annotation says to call this method when the HTTP PUT method is called. The `Consumes` annotation tells it that the method expects JSON to be received. The method takes a `JsonObject`. It calls the `MeetingManager` service to add the meeting and then returns a 201 Created response with a link to the created resource.

```java
@PUT
@Consumes(MediaType.APPLICATION_JSON)
public Response add(JsonObject m) {
    manager.add(m);
    UriBuilder builder = info.getBaseUriBuilder();
    builder.path(MeetingService.class).path(m.getString("id"));
    return Response.created(builder.build()).build();
}
```

 * The method introduces several new classes that need to be imported: `PUT` and `Consumes` are in package `javax.ws.rs`; `Response`, `MediaType`, and `UriBuilder` are all in `javax.ws.rs.core`; `JsonObject` is in the `javax.json package`. Take care when importing `MediaType` and `Response` as Java contains multiple classes with these names:

```java
import javax.ws.rs.Consumes;
import javax.ws.rs.PUT;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import javax.ws.rs.core.UriBuilder;
import javax.json.JsonObject;
```

2. A method to list all the meetings that exist is very simple. The `GET` annotation is used to indicate this will be called on an HTTP GET request and the `Produces` method indicates that JSON will be returned to the client:

```java
@GET
@Produces(MediaType.APPLICATION_JSON)
public JsonArray list() {
    return manager.list();
}
```

 * This method introduces two new classes that need to be imported: Both `GET` and `Produces` are in the `javax.ws.rs` package. Take care when importing `Produces` as Java EE contains multiple classes with this name:

```java
import javax.ws.rs.GET;
import javax.ws.rs.Produces;
import javax.json.JsonArray;
```

3. To get the details of a single meeting we need a method that responds to a child path of the path provided on the class definition. This can be done using the `Path` annotation on the method. The `Path` value can contain either a literal or, in this case, a named entity that can then be passed in as a method parameter. The `PathParam` annotation is used on the method parameter to indicate which part of the path this parameter should be provided from.

```java
@GET
@Path("{id}")
@Produces(MediaType.APPLICATION_JSON)
public JsonObject get(@PathParam("id") String id) {
    return manager.get(id);
}
```

 * This method introduces one new class that needs to be imported: `PathParam` is in the `javax.ws.rs` package. Take care when importing `PathParam` as Java EE contains multiple classes with this name:

```java
import javax.ws.rs.PathParam;
```

4. Finally, you need to write a method that starts the meeting. This method responds to an HTTP post which is indicated using the `POST` annotation. In this case it responds to a specific resource instance and, to ensure that the `JsonObject` and the path don’t have conflicting information, the JsonObject’s ID in the `JsonObject` is overwritten by the one from the path. In this application it is not important but if there were a security constraint on the URL, it could be crucial.

```java
@POST
@Path("{id}")
@Consumes(MediaType.APPLICATION_JSON) 
public void startMeeting(@PathParam("id") String id, JsonObject m){
    JsonObjectBuilder builder = MeetingsUtil.createJsonFrom(m);
    builder.add("id", id);
    manager.startMeeting(builder.build());
}
```

 * This method introduces one new class which needs to be imported: `POST` is in the `javax.ws.rs.package`:

```java
import javax.ws.rs.POST;
import javax.json.JsonObjectBuilder;
```

5. Save the file.

### Step 4. Creating the MeetingApplication class

The last step is to tell JAX-RS that this module should be treated as a JAX-RS application. There are a few ways to do this but the simplest is with a class:

 * Create a new class called `MeetingApplication` with the superclass `javax.ws.rs.core.Application`.

When the class opens in the editor, add an annotation to tell the JAX-RS annotation where to dispatch requests to REST endpoints from:

1. Before the class definition add the `@ApplicationPath` annotation with a value of `"/rest/"`:
```java
import javax.ws.rs.ApplicationPath;
 
@ApplicationPath("/rest/")
```

2. Save the file.

The application is done and you are ready to run it.

You can check that you’ve copied the code correctly by comparing the classes against the code in GitHub on the Master branch of the repository.

In Eclipse, you might see some warnings, which you can ignore. For example, the HTML problems are because Eclipse doesn’t understand the AngularJS tags which are used to define the application’s UI.

### Step 5. Running the application
#### Eclipse WDT
There are two ways to get the application running from within WDT:

 * The first is to use Maven to build and run the project:
 1. Run the Maven `install` goal to build and test the project: Right-click **pom.xml** in the `meetings` project, click **Run As… > Maven Build…**, then in the **Goals** field type `install` and click **Run**. The first time you run this goal, it might take a few minutes to download the Liberty dependencies.
 2. Run a Maven build for the `liberty:start-server goal`: Right-click **pom.xml**, click **Run As… > Maven Build...**, then in the **Goals** field, type `liberty:start-server` and click **Run**. This starts the server in the background.
 3. Open the application, which is available at `http://localhost:9080/meetings/`.
 4. To stop the server again, run the `liberty:stop-server` build goal.

 * The second way is to right-click the `meetings` project and select **Run As… > Run on Server** but there are a few things to note if you do this. WDT doesn’t automatically add the MicroProfile features as you would expect so you need to manually add those. Also, any changes to the configuration in `src/main/liberty/config` won’t be picked up unless you add an include.

Find out more about [MicroProfile and WebSphere Liberty](https://developer.ibm.com/wasdev/docs/microprofile/).

## Next Steps
Part 2: [MicroProfile Meeting Application - Adding Persistance](https://github.com/IBM/microprofile-meeting-persistance)
