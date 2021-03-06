# E-Commerce Recommendation Template with rate event as training data.

This examples demonstrates how to modify E-Commerce Recommendation template to use "rate" event as Training Data.

However, recent "view" event is still used for recommendation for new user (to recommend items similar to what new user just recently viewed) and the returned scores are not predicted rating but a ranked scores for new user.

This template also supports that the user may rate same item multiple times and latest rating value will be used for training. The modification can be further simplified if the support of this case is not needed.

The modification is based on E-Commerce Recommendation template v0.1.1.

You can find the complete modified source code in `src/` directory.

## Documentation

Please refer to http://docs.prediction.io/templates/ecommercerecommendation/quickstart/

### import sample data

```
$ python data/import_eventserver.py --access_key <your_access_key>
```


## Modification

### DataSource.scala

In DataSource, change ViewEvent case class to RateEvent. Add `rating: Double` is added to the RateEvent.

```scala
// MODIFIED
case class RateEvent(user: String, item: String, rating: Double, t: Long)
```

Modify TrainingData class to use rateEvent

```scala
class TrainingData(
  val users: RDD[(String, User)],
  val items: RDD[(String, Item)],
  val rateEvents: RDD[RateEvent] // MODIFIED
) extends Serializable {
  override def toString = {
    s"users: [${users.count()} (${users.take(2).toList}...)]" +
    s"items: [${items.count()} (${items.take(2).toList}...)]" +
    // MODIFIED
    s"rateEvents: [${rateEvents.count()}] (${rateEvents.take(2).toList}...)"
  }
}
```

Modify `readTraining()` function to read "rate" events (commented with "// MODIFIED"). Replace all `ViewEvent` with `RateEvent`. Replace all `viewEvent` with `rateEvent`. Retrieve the rating value from the event properties:

```scala

    // get all "user" "rate" "item" events
    val rateEventsRDD: RDD[RateEvent] = eventsDb.find( // MODIFIED
      appId = dsp.appId,
      entityType = Some("user"),
      eventNames = Some(List("rate")), // MODIFIED
      // targetEntityType is optional field of an event.
      targetEntityType = Some(Some("item")))(sc)
      // eventsDb.find() returns RDD[Event]
      .map { event =>
        val rateEvent = try {
          event.event match {
            case "rate" => RateEvent( // MODIFIED
              user = event.entityId,
              item = event.targetEntityId.get,
              rating = event.properties.get[Double]("rating"), // ADDED
              t = event.eventTime.getMillis)
            case _ => throw new Exception(s"Unexpected event ${event} is read.")
          }
        } catch {
          case e: Exception => {
            logger.error(s"Cannot convert ${event} to RateEvent." + // MODIFIED
              s" Exception: ${e}.")
            throw e
          }
        }
        rateEvent
      }.cache()

    new TrainingData(
      users = usersRDD,
      items = itemsRDD,
      rateEvents = rateEventsRDD // MODIFIED
    )

```

### Prepartar.scala

Modify Preparator to pass rateEvents to algorithm as PreparedData (Replace all `ViewEvent` with `RateEvent`. Replace all `viewEvent` with `rateEvent`)

```scala
  ...
  new PreparedData(
    users = trainingData.users,
    items = trainingData.items,
    rateEvents = trainingData.rateEvents) // MODIFIED

  ...

class PreparedData(
  val users: RDD[(String, User)],
  val items: RDD[(String, Item)],
  val rateEvents: RDD[RateEvent] // MODIFIED
) extends Serializable

```

### ALSAlgorithm.scala

Modify `train()` method to train with rate event.

```scala

  def train(sc: SparkContext, data: PreparedData): ALSModel = {
    require(!data.rateEvents.take(1).isEmpty, // MODIFIED
      s"rateEvents in PreparedData cannot be empty." +
      " Please check if DataSource generates TrainingData" +
      " and Preprator generates PreparedData correctly.")

    ...

    val mllibRatings = data.rateEvents // MODIFIED
      .map { r =>
        ...

        ((uindex, iindex), (r.rating, r.t)) // MODIFIED
      }.filter { case ((u, i), v) =>
        // keep events with valid user and item index
        (u != -1) && (i != -1)
      }.reduceByKey { case (v1, v2) => // MODIFIED
        // if a user may rate same item with different value at different times,
        // use the latest value for this case.
        // Can remove this reduceByKey() if no need to support this case.
        val (rating1, t1) = v1
        val (rating2, t2) = v2
        // keep the latest value
        if (t1 > t2) v1 else v2
      }
      .map { case ((u, i), (rating, t)) => // MODIFIED
        // MLlibRating requires integer index for user and item
        MLlibRating(u, i, rating) // MODIFIED
      }.cache()

    ...
  }
```

Change from `ALS.trainImplicit()` to `ALS.train()` for training with explicit rating data.

```scala
    ...
    val m = ALS.train( // MODIFIED
      ratings = mllibRatings,
      rank = ap.rank,
      iterations = ap.numIterations,
      lambda = ap.lambda,
      blocks = -1,
      seed = seed)

    ...  
```
