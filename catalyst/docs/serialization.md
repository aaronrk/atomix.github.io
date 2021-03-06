---
layout: docs
project: catalyst
menu: docs
title: Serialization
pitch: Custom binary serialization, built for the JVM
first-section: serialization
---

Catalyst provides an efficient custom serialization framework that's designed to operate on both disk and memory via a common [`Buffer`](#buffers) abstraction.

## Serializer

Catalyst's serializer can be used by simply instantiating a [`Serializer`][Serializer] instance:

```java
// Create a new Serializer instance with an unpooled heap allocator
Serializer serializer = new Serializer(new UnpooledHeapAllocator());

// Register the Person class with a serialization ID of 1
serializer.register(Person.class, 1);
```

Objects are serialized and deserialized using the `writeObject` and `readObject` methods respectively:

```java
// Create a new Person object
Person person = new Person(1234, "Jordan", "Halterman");

// Write the Person object to a newly allocated buffer
Buffer buffer = serializer.writeObject(person);

// Flip the buffer for reading
buffer.flip();

// Read the Person object
Person result = serializer.readObject(buffer);
```

By default, the [`Serializer`][Serializer] class supports serialization and deserialization of [`CatalystSerializable`][CatalystSerializable] types, types that have an associated [`Serializer`][Serializer], and native Java [`Serializable`][Serializable] and [`Externalizable`][Externalizable] types, with [`Serializable`][Serializable] being the most inefficient method of serialization.

Additionally, Catalyst support copying objects by serializing and deserializing them. To copy an object, simply use the [`copy`][Serializer.copy] method:

```java
Person copy = serializer.copy(person);
```

## Type Serializers

Custom type serializers can be registered on a [`Serializer`][Serializer] instance by either passing a [`SerializableTypeResolver`][SerializableTypeResolver] to the [`Serializer`][Serializer] constructor or using the [`register`][Serializer.register] method.

By default, Catalyst registers serializable types provided by [`PrimitiveTypeResolver`][PrimitiveTypeResolver] and [`JdkTypeResolver`][JdkTypeResolver], including the following types:

* Primitive types
* Primitive wrappers
* Primitive arrays
* Primitive wrapper arrays
* `String`
* `Class`
* `BigInteger`
* `BigDecimal`
* `Date`
* `Calendar`
* `TimeZone`
* `Map`
* `List`
* `Set`

Users can resolve custom serializers at runtime via [`resolve`][Serializer.resolve] methods or register specific types via [`register`][Serializer.register] methods.

To register a serializable type with an [`Serializer`][Serializer] instance, the type must generally meet one of the following conditions:

* Implement [`CatalystSerializable`][CatalystSerializable]
* Implement [`Externalizable`][Externalizable]
* Provide a [`TypeSerializer`][TypeSerializer] class
* Provide a [`TypeSerializerFactory`][TypeSerializerFactory]

```java
Serializer serializer = new Serializer();
serializer.register(Foo.class, FooSerializer.class);
serializer.register(Bar.class);
```

At the core of the serialization framework is the [`TypeSerializer`][TypeSerializer]. The [`TypeSerializer`][TypeSerializer] is a simple interface that exposes two methods for serializing and deserializing objects of a specific type respectively. That is, serializers are responsible for serializing objects of other types, and not themselves. Catalyst provides this separate serialization interface in order to allow users to create custom serializers for types that couldn't otherwise be serialized by Catalyst.

The [`TypeSerializer`][TypeSerializer] interface consists of two methods:

```java
public class FooSerializer implements TypeSerializer<Foo> {
  @Override
  public void write(Foo foo, BufferWriter writer, Serializer serializer) {
    writer.writeInt(foo.getBar());
  }

  @Override
  @SuppressWarnings("unchecked")
  public Foo read(Class<Foo> type, BufferReader reader, Serializer serializer) {
    Foo foo = new Foo();
    foo.setBar(reader.readInt());
  }
}
```

To serialize and deserialize an object, simply write to and read from the passed in [`BufferOutput`][BufferOutput] or [`BufferInput`][BufferInput] instance respectively. In addition to the reader/writer, the `Serializer` that is serializing or deserializing the instance is also passed in. This allows the serializer to serialize or deserialize subtypes as well:

```java
public class FooSerializer implements TypeSerializer<Foo> {
  @Override
  public void write(Foo foo, BufferOutput output, Serializer serializer) {
    output.writeInt(foo.getBar());
    Baz baz = foo.getBaz();
    serializer.writeObject(baz, output);
  }

  @Override
  @SuppressWarnings("unchecked")
  public Foo read(Class<Foo> type, BufferInput input, Serializer serializer) {
    Foo foo = new Foo();
    foo.setBar(input.readInt());
    foo.setBaz(serializer.readObject(input));
  }
}
```

Catalyst comes with a number of native [`TypeSerializer`][TypeSerializer] implementations, for instance `ArrayListSerializer`:

```java
public class ArrayListSerializer implements TypeSerializer<ArrayList> {
  @Override
  public void write(ArrayList object, BufferOutput output, Serializer serializer) {
    output.writeUnsignedShort(object.size());
    for (Object value : object) {
      serializer.writeObject(value, output);
    }
  }

  @Override
  @SuppressWarnings("unchecked")
  public ArrayList read(Class<ArrayList> type, BufferInput input, Serializer serializer) {
    int size = input.readUnsignedShort();
    ArrayList object = new ArrayList<>(size);
    for (int i = 0; i < size; i++) {
      object.add(serializer.readObject(input));
    }
    return object;
  }
}
```

## Serializable Type Identifiers

Types explicitly registered with a [`Serializer`][Serializer] instance can provide a registration ID in lieu of serializing class names. If given a serialization ID, Catalyst will write the serializable type ID to the serialized [`Buffer`][Buffer] instance of the class name and use the ID to locate the serializable type upon deserializing the object. This means *it is critical that all processes that register a serializable type use consistent identifiers.*

To register a serializable type ID, pass the `id` to the [`register`][Serializer.register] method:

```java
Serializer serializer = new Serializer();
serializer.register(Foo.class, 1, FooSerializer.class);
serializer.register(Bar.class, 2);
```

Valid serialization IDs are between `0` and `65535`. However, Catalyst reserves IDs `128` through `255` for internal use. Attempts to register serializable types within the reserved range will result in an `IllegalArgumentException`. If no serializable type ID is provided, Catalyst will hash the serializable type name to create a type ID. This means types explicitly registered on a [`Serializer`][Serializer] used to serialize an object must also be registered on any serializer that deserializes the same object.

## Abstract Serializers

Because Java is an object oriented language, in many cases users may want to be able to deserialize an interface to a specific implementation. For example, a `List` object could be deserialized to a number of different implementations. Catalyst allows users to register serializers for abstract types like interfaces and abstract base classes. To register an abstract serializer, use the [`registerAbstract`][Serializer.registerAbstract] method:

```java
serializer.registerAbstract(Map.class, 1, HashMapSerializer.class);
```

Registering an abstract serializer will allow all types that extend the abstract type to be serialized and deserialized by Catalyst. If a concrete type that extends an abstract type is registered, the concrete type will supercede the abstract type.

```java
serializer.register(TreeMap.class, 2, TreeMapSerializer.class);
```

In the example above, if a `TreeMap` is serialized, Catalyst will use the `TreeMapSerializer`, but any other type of `Map` will be serialized and deserialized with the `HashMapSerializer`.

## Default Serializers

Default serializers are used to serialize types for which no specific [`TypeSerializer`][TypeSerializer] is provided. When a serializable type is registered without a [`TypeSerializer`][TypeSerializer], the first default serializer found for the given type is assigned as the serializer for that type. Default serializers are evaluated against registered types in reverse insertion order, so default serializers registered more recently take precedence over default serializers registered earlier.

The use case for default serializers is registering serialization frameworks in Catalyst. For example, the `Serializable` interface is an identifier interface that relates only to serialization rather than to the type of an object. `Serializable` can be applied to many different types of classes. Registering a default `Serializable` serializer allows Catalyst to fall back to Java serialization when no serializer has been explicitly registered for the type.

To register a default serializer, use the [`registerDefault`][Serializer.registerDefault] methods:

```java
serializer.registerDefault(Serializable.class, JavaSerializableSerializer.class);
```

## Serialization Frameworks

In addition to support for Java serialization, Catalyst provides generic default serializers based on [Kryo](https://github.com/EsotericSoftware/kryo) and [Jackson](https://github.com/FasterXML/jackson). To use those serialization frameworks, add the appropriate Maven artifact to your project and register the default serializer:

```java
serializer.registerDefault(KryoSerializable.class, GenericKryoSerializer.class);
```

## CatalystSerializable

Instead of writing a custom [`TypeSerializer`][TypeSerializer], serializable types can also implement the [`CatalystSerializable`][CatalystSerializable] interface. The [`CatalystSerializable`][CatalystSerializable] interface is synonymous with Java's native `Serializable` interface. As with the [`TypeSerializer`][TypeSerializer] interface, [`CatalystSerializable`][CatalystSerializable] exposes two methods which receive both a [`Buffer`](#buffers) and a [`Serializer`][Serializer]:

```java
public class Foo implements CatalystSerializable {
  private int bar;
  private Baz baz;

  public Foo() {
  }

  public Foo(int bar, Baz baz) {
    this.bar = bar;
    this.baz = baz;
  }

  @Override
  public void writeObject(BufferOutput buffer, Serializer serializer) {
    buffer.writeInt(bar);
    serializer.writeObject(baz);
  }

  @Override
  public void readObject(BufferInput buffer, Serializer serializer) {
    bar = buffer.readInt();
    baz = serializer.readObject(buffer);
  }
}
```

## Pooled Object Deserialization

Catalyst's serialization framework integrates with [object pools](#buffer-pools) to support allocating pooled objects during deserialization. When a [`Serializer`][Serializer] instance is used to deserialize a type that implements `ReferenceCounted`, Catalyst will automatically create new objects from a `ReferencePool`:

```java
Serializer serializer = new Serializer();

// Person implements ReferenceCounted<Person>
Person person = serializer.readObject(buffer);

// ...do some stuff with Person...

// Release the Person reference back to Catalyst's internal Person pool
person.release();
```

{% include common-links.html %}