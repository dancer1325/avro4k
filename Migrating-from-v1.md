* goal
  * how to migrate from Avro4k v1 -- to -- v2

## Pure avro serialization

```kotlin
// PREVIOUSLY
val bytes = Avro.default.encodeToByteArray(TheDataClass.serializer(), TheDataClass(...))
Avro.default.decodeFromByteArray(TheDataClass.serializer(), bytes)

// NOW
val bytes = Avro.encodeToByteArray(TheDataClass(...))
Avro.decodeFromByteArray<TheDataClass>(bytes)
```

## Set a field default value to null

```kotlin
// PREVIOUSLY
data class TheDataClass(
    @AvroDefault(Avro.NULL)
    val field: String?
)

// NOW
data class TheDataClass(
    // ... Nothing, as it is the default behavior!
    val field: String?
)

// Or
val avro = Avro { implicitNulls = false }
data class TheDataClass(
    @AvroDefault("null")
    val field: String?
)
```

## Set a field default value to empty array

```kotlin
// PREVIOUSLY
data class TheDataClass(
    @AvroDefault("[]")
    val field: List<String>
)

// NOW
data class TheDataClass(
    // ... Nothing, as it is the default behavior!
    val field: List<String>
)

// Or
val avro = Avro { implicitEmptyCollections = false }
data class TheDataClass(
    @AvroDefault("[]")
    val field: List<String>
)
```

## Set a field default value to empty map

```kotlin
// PREVIOUSLY
data class TheDataClass(
    @AvroDefault("{}")
    val field: Map<String, String>
)

// NOW
data class TheDataClass(
    // ... Nothing, as it is the default behavior!
    val field: Map<String, String>
)

// Or
val avro = Avro { implicitEmptyCollections = false }
data class TheDataClass(
    @AvroDefault("{}")
    val field: Map<String, String>
)
```

## generic data serialization
Convert a kotlin data class to a `GenericRecord` to then be handled by a `GenericDatumWriter` in avro.

```kotlin
// PREVIOUSLY
val genericRecord: GenericRecord = Avro.default.toRecord(TheDataClass.serializer(), TheDataClass(...))
Avro.default.fromRecord(TheDataClass.serializer(), genericRecord)

// NOW
val genericData: Any? = Avro.encodeToGenericData(TheDataClass(...))
Avro.decodeFromGenericData<TheDataClass>(genericData)
```

## Configure the `Avro` instance

```kotlin
// PREVIOUSLY
val avro = Avro(
    AvroConfiguration(
        namingStrategy = FieldNamingStrategy.SnackCase,
        implicitNulls = true,
    ),
    SerializersModule {
         contextual(CustomSerializer())
    }
)

// NOW
val avro = Avro {
    namingStrategy = FieldNamingStrategy.SnackCase
    implicitNulls = true
    serializersModule = SerializersModule {
         contextual(CustomSerializer())
    }
}
```

## Changing the name of a record

```kotlin
// PREVIOUSLY
@AvroName("TheName")
@AvroNamespace("a.custom.namespace")
data class TheDataClass(...)

// NOW
@SerialName("a.custom.namespace.TheName")
data class TheDataClass(...)
```

## Writing an avro object container file with a custom field naming strategy

```kotlin
// PREVIOUSLY
Files.newOutputStream(Path("/your/file.avro")).use { outputStream ->
    Avro(AvroConfiguration(namingStrategy = SnakeCaseNamingStrategy))
        .openOutputStream(TheDataClass.serializer()) { encodeFormat = AvroEncodeFormat.Data(CodecFactory.snappyCodec()) }
        .to(outputStream)
        .write(TheDataClass(...))
        .write(TheDataClass(...))
        .write(TheDataClass(...))
        .close()
}


// NOW
val dataSequence = sequenceOf(
    TheDataClass(...),
    TheDataClass(...),
    TheDataClass(...),
)
Files.newOutputStream(Path("/your/file.avro")).use { outputStream ->
    val writer = AvroObjectContainer { fieldNamingStrategy = FieldNamingStrategy.SnakeCase }
        .openWriter(outputStream) {
            codec(CodecFactory.snappyCodec())
            // you can also add your metadata !
            metadata("myProp", 1234L)
            metadata("a string metadata", "hello")
        }
    writer.use {
        dataSequence.forEach { writer.write(it) }
    }
}
```

## Reading PREVIOUSLY written binary encoded (`AvroEncodeFormat.Binary`) files

It was possible to use the `AvroEncodeFormat.Binary` format, which used [binary encoding](https://avro.apache.org/docs/current/spec.html#binary_encoding). The data did not have an embedded schema, so it had to be specified when reading.

```kotlin
// PREVIOUSLY
Avro.default.openInputStream(serializer) { decodeFormat = AvroDecodeFormat.Binary(schema) }
    .from(data).use { avroInputStream -> return avroInputStream.nextOrThrow() }

// NOW
val inputStream = ByteArrayInputStream(data)
while (inputStream.remaining() > 0) {
    // If the writer schema corresponds to the specified type
    val element = Avro.decodeFromStream<MyType>(inputStream)

    // If the writer schema does not correspond to the specified type
    val element = Avro.decodeFromStream<MyType>(writerSchema, inputStream)

    // With explicit writer schema and serializer
    val element = Avro.decodeFromStream(writerSchema, serializer, inputStream)
}
```

## Writing a collection of Byte as BYTES or FIXED

If you really want to encode a `BYTES` type, just use the `ByteArray` type or write your own `AvroSerializer` to control the schema and its serialization.

For encoding to `FIXED`, then just use the `ByteArray` type with the `AvroFixed` annotation (or still write your own serializer).

```kotlin
// PREVIOUSLY
@Serializable
data class TheDataClass(
    val collectionOfBytes: List<Byte>,
    val listOfBytes: List<Byte>,
    val setOfBytes: List<Byte>,
)

// NOW
@Serializable
data class TheDataClass(
    val collectionOfBytes: ByteArray,
    val listOfBytes: ByteArray,
    val setOfBytes: ByteArray,
)
```

## Serialize a `BigDecimal` as a string

> [!INFO]
> Note that you can replace `@Serializable(with = BigDecimalAsStringSerializer::class)` with `@Contextual` to use the default global `BigDecimalSerializer` already registered,
> which is already compatible with the `@AvroStringable` feature.

```kotlin
// PREVIOUSLY
@Serializable
data class MyData(
    @Serializable(with = BigDecimalAsStringSerializer::class)
    val bigDecimalAsString: BigDecimal,
)

// NOW
@Serializable
data class MyData(
    @Contextual
    @AvroStringable
    val bigDecimalAsString: BigDecimal,
)
```

## Serialize a `BigDecimal` as a BYTES or FIXED

PREVIOUSLY, a BigDecimal was serialized as bytes with 2 as scale and 8 as precision. NOW you have to explicitly declare the needed scale and precision using `@AvroDecimal`, 
or use `@AvroStringable` to serialize it as a string which doesn't need scale nor precision.

> [!INFO]
> Note that you can replace `@Serializable(with = BigDecimalSerializer::class)` with `@Contextual` to use the default global `BigDecimalSerializer` already registered.

```kotlin
// PREVIOUSLY
@Serializable
data class MyData(
    @Serializable(with = BigDecimalSerializer::class)
    val bigDecimalAsBytes: BigDecimal,
    @AvroFixed(10)
    @Serializable(with = BigDecimalSerializer::class)
    val bigDecimalAsFixed: BigDecimal,
)

// NOW
@Serializable
data class MyData(
    @Contextual
    @AvroDecimal(scale = 8, precision = 2)
    val bigDecimalAsBytes: BigDecimal,
    @Contextual
    @AvroFixed(10)
    @AvroDecimal(scale = 8, precision = 2)
    val bigDecimalAsFixed: BigDecimal,
)
```
