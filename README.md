# rejoiner能够从gRPC微服务和其他Protobuf源生成统一的GraphQL schema，具有以下功能：
  - 从微服务创建统一的GraphQL模式
  - 可灵活定义GraphQL模式并组成共享组件
  - 从Proto定义生成GraphQL类型
  - 基于GraphQL查询参数填充请求Proto
  - 提供一个DSL来修改生成的模式
  - 通过注释获取数据的方法来加入数据源
  - 基于GraphQL选择器创建Proto FieldMasks
  
# Rejoiner

 - Creates a uniform GraphQL schema from microservices
 - Allows the GraphQL schema to be flexibly defined and composed as shared components
 - Generates GraphQL types from Proto definitions
 - Populates request Proto based on GraphQL query parameters
 - Supplies a DSL to modify the generated schema
 - Joins data sources by annotating methods that fetch data
 - Creates Proto [FieldMasks](https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/FieldMask) based on GraphQL selectors

 ![Rejoiner Overview](./website/static/rejoiner_overview.svg)

## Experimental Features

These features are actively being developed.

 - Relay support [[Example](./examples/java/com/google/api/graphql/examples/library)]
 - GraphQL Stream (based on gRPC streaming) [[Example](./examples/java/com/google/api/graphql/examples/streaming)]

## Schema Module

SchemaModule is a Guice module that is used to generate parts of a GraphQL
schema. It finds methods and fields that have Rejoiner annotations when it's
instantiated. It then looks at the parameters and return type of these methods
in order to generate the appropriate GraphQL schema. Examples of queries,
mutations, and schema modifications are presented below.

## GraphQL Query

```java
final class TodoQuerySchemaModule extends SchemaModule {
  @Query("listTodo")
  ListenableFuture<ListTodoResponse> listTodo(ListTodoRequest request, TodoClient todoClient) {
    return todoClient.listTodo(request);
  }
}
```

In this example `request` is of type `ListTodoRequest` (a protobuf message), so
it's used as a parameter in the generated GraphQL query. `todoService` isn't a
protobuf message, so it's provided by the Guice injector.

This is useful for providing rpc services or database access objects for
fetching data. Authentication data can also be provided here.

Common implementations for these annotated methods:
 - Make gRPC calls to microservices which can be implemented in any language
 - Load protobuf messages directly from storage
 - Perform arbitrary logic to produce the result

## GraphQL Mutation

```java
final class TodoMutationSchemaModule extends SchemaModule {
  @Mutation("createTodo")
  ListenableFuture<Todo> createTodo(
      CreateTodoRequest request, TodoService todoService, @AuthenticatedUser String email) {
    return todoService.createTodo(request, email);
  }
}
```

## Adding edges between GraphQL types

In this example we are adding a reference to the User type on the Todo type.
```java
final class TodoToUserSchemaModule extends SchemaModule {
  @SchemaModification(addField = "creator", onType = Todo.class)
  ListenableFuture<User> todoCreatorToUser(UserService userService, Todo todo) {
    return userService.getUserByEmail(todo.getCreatorEmail());
  }
}
```
In this case the Todo parameter is the parent object which can be referenced to
get the creator's email.

This is how types are joined within and across APIs.

![Rejoiner API Joining](./website/static/rejoiner.svg)

## Removing a field

```java
final class TodoModificationsSchemaModule extends SchemaModule {
  @SchemaModification
  TypeModification removePrivateTodoData =
      Type.find(Todo.getDescriptor()).removeField("privateTodoData");
}
```

## Building the GraphQL schema
```java
import com.google.api.graphql.rejoiner.SchemaProviderModule;

public final class TodoModule extends AbstractModule {
  @Override
  protected void configure() {
    // Guice module that provides the generated GraphQLSchema instance
    install(new SchemaProviderModule());

    // Install schema modules
    install(new TodoQuerySchemaModule());
    install(new TodoMutationSchemaModule());
    install(new TodoModificationsSchemaModule());
    install(new TodoToUserSchemaModule());
  }
}
```

## Getting started

Currently we are only publishing SNAPSHOT builds to sonatype.

Here is an example build.gradle file, also see the examples directory for a
complete example.

```
  repositories {
    // ...
    maven {
      url 'https://oss.sonatype.org/content/repositories/snapshots/'
    }
  }

  // ...
  compile "com.google.api.graphql:rejoiner:0.0.1-SNAPSHOT"
  compile "com.google.api.graphql:execution:0.0.1-SNAPSHOT"
```

## Supported return types

All generated proto messages extend `Message`.
 - Any subclass of `Message`
 - `ImmutableList<? extends Message>`
 - `ListenableFuture<? extends Message>`
 - `ListenableFuture<ImmutableList<? extends Message>>`

## Project information

 - Rejoiner is built on top of [GraphQL-Java](https://github.com/graphql-java/graphql-java) which provides the core
   GraphQL capabilities such as query parsing, validation, and execution.  
 - Java code is formatted using [google-java-format](https://github.com/google/google-java-format).
 - Note: This is not an official Google product.
