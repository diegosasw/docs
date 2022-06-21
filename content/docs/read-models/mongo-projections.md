---
title: "MongoDB projections"
description: "Projecting events to MongoDB documents"
weight: 630
---

MongoDB is a popular document database, and Eventuous natively supports projecting events to it. In combination with Mongo [checkpoint store]({{< ref "checkpoint" >}}) you can comfortably use MongoDB as a queryable store for your read models.

The base class for a MongoDB projection is `MongoProjection<T>` where `T` is a record derived from the `ProjectedDocument` abstract record. 

## Projected document

The `ProjectedDocument` record has two properties: `Id`, which is used as the document id, and `Position`. The `Position` property is set by the Mongo projection implicitly when an event is projected to a document. The value set for this property is the projection position in the subscribed stream. You can use this information for addressing the [consistency concern]({{< ref "rm-concept#dealing-with-stale-data" >}}).

Here is an example of a document model from the sample application:

```csharp
public record BookingDocument : ProjectedDocument {
    public BookingDocument(string id) : base(id) { }

    public string    GuestId      { get; init; }
    public string    RoomId       { get; init; }
    public LocalDate CheckInDate  { get; init; }
    public LocalDate CheckOutDate { get; init; }
    public float     BookingPrice { get; init; }
    public float     PaidAmount   { get; init; }
    public float     Outstanding  { get; init; }
    public bool      Paid         { get; init; }
}
```

You might notice that it mostly replicates the `Booking` aggregate state, and that's not the best practice, but it will serve the purpose as an example.

## Dependencies

The only dependency for a MongoDB projection is `IMongoDatabase` instance. One projection is expected to work with one document collection, as it is the default for mapping document models in C# to BSON models in MongoDB. The collection name is derived from the document model class, excluding the `Document` part. For example, the aforementioned `BookingDocument` model would be mapped to the `booking` collection.

The dependency needs to be passed to the base class constructor, otherwise the code won't compile. Here is the constructor of a sample projection that uses the `BookingDocument` model:

```csharp
public class BookingStateProjection : MongoProjection<BookingDocument> {
    public BookingStateProjection(IMongoDatabase database) : base(database) { 
        // projectors will be defined here
    }
}
```

## Projecting events

We call the functions that project individual events _projectors_. One projection class can have multiple projectors, one per event type. You cannot define more than one projector for the same event type.

Projectors must be registered in the projection constructor using the `On<TEvent>` method and its overloads, where `TEvent` is the event type. The base `On<TEvent>` accepts the `ProjectTypedEvent<T, TEvent>` function instance where the delegate signature is defined like this:

```csharp
public delegate ValueTask<MongoProjectOperation<T>> ProjectTypedEvent<T, TEvent>(
    MessageConsumeContext<TEvent> consumeContext
) where T : ProjectedDocument where TEvent : class;
```

The function must return a value task with an instance of `MongoProjectOperation<T>` where `T` is the document model. This type is a record type defined as:

```csharp
public record MongoProjectOperation<T>(Func<IMongoCollection<T>, CancellationToken, Task> Execute);
```

As you can see, a `MongoProjectOperation` record has a single property that is a function, which receives the MongoDB collection, a cancellation token, and returns a `Task`, so it is an asynchronous function.

It all sounds a bit like a Russian doll, but Eventuous provides a few helpful ways to simplify this.

### Operation builders

Since version 0.9.0, Eventuous allows to build an instance of `ProjectTypedEvent` function using operation builders. You can use an overload of `On<TEvent>` that accepts the builder configuration function to specify all the basic MongoDB operations supported by the Mongo C# driver. It starts by specifying what operation you want to execute (insert one or many, update one or many, etc):

```csharp
On<V1.BookingCancelled>(
    b => b.UpdateOne
    // more to follow
);
```

Then, you need to add the necessary operation details. For example, for updating one or many documents you need a filter and an update definition:

```csharp
On<V1.BookingCancelled>(
    b => b.UpdateOne
        .Filter((evt, doc) =>
            doc.Bookings.Select(booking => booking.BookingId).Contains(evt.BookingId)
        )
        .Update((evt, update) =>
            update.PullFilter(
                x => x.Bookings,
                x => x.BookingId == evt.BookingId
            )
        )
);
```

There are two flavours of the `Filter` function:
- Use the native `FilterDefinitionBuilder`
- Use the function that returns a boolean as shown in the example above

Filters are required for `UndateOne`, `UpdateMany`, `DeleteOne` and `DeleteMany` operations.

When you use `UpdateOne` or `DeleteOne` you can simplify the filter further by using the `Id` function instead of `Filter`. There, you just need to provide a function to get the document id from the event:

```csharp
On<V1.BookingCancelled>(
    b => b.UpdateOne
        .Id(evt => evt.BookingId)
        .Update(...)
);
```

The `Id` function will build a filter for the document identity to be equal the given value from the event.

Naturally, for deletions you don't need to specify anything else that the filter using either one of the `Filter` functions or the `Id` function (for `DeleteOne`).

Both `UpdateOne` and `UpdateMany` requires to specify the update itself. There are two flavours of the `Update` function that you can use: synchronous and asynchronous. The synchronous one is the most frequently used, and it is shown in the example above. It only uses the values from the event to build an update operation, and we recommend doing that only. However, you might get into a situation when you need to get data from somewhere else. In that case, you can use the asynchronous version:

```csharp
On<V1.BookingCancelled>(
    b => b.UpdateOne
        .Id(evt => evt.BookingId)
        .Update(async (evt, update) =>
            var missingData = await GetDataFromElsewhere();
            update.Set(x => x.Somedata, missingData);
        )
);
```

For inserting documents we recommend using updates, and the default update option is set to allow upserts. This way, you don't need to care if you are accidentally (or intentionally) replay historical events over the existing collection.

You can still use the insert operation using `InsertOne` or `InsertMany` builders. In that case, you don't need a filter not an update. All you need is to provide a way to create a projected document instance from the event using the `Document` function. There are, again, two flavours of it - synchronous (preferred) and asynchronous (to get more data elsewhere). For example:

```csharp
On<V1.RoomBooked>(b => b.InsertOne.Document(evt => new BookingDocument(...)));
```

For `InsertMany` operation you need to use the `Documents` function that should return a list of documents to insert.

Finally, you can configure each of those operations by using the `Configure` function. It receives the options instance for each operation (`InsertOneOptions`, `InsertManyOptions`, etc) and can change its properties. In most cases, Eventuous uses the default options. However, as mentioned previously, update options are configured to allow upserts.
