# Лабораторная 3. Потоковая обработка в Apache Flink

## RideCleansingExercise
	public class RideCleansingExercise extends ExerciseBase {
	public static void main(String[] args) throws Exception {

		ParameterTool params = ParameterTool.fromArgs(args);
		final String input = params.get("input", ExerciseBase.pathToRideData);

		final int maxEventDelay = 60;       // events are out of order by max 60 seconds
		final int servingSpeedFactor = 600; // events of 10 minutes are served in 1 second

		// set up streaming execution environment
		StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		env.setParallelism(ExerciseBase.parallelism);

		// start the data generator
		DataStream<TaxiRide> rides = env.addSource(rideSourceOrTest(new TaxiRideSource(input, maxEventDelay, servingSpeedFactor)));

		DataStream<TaxiRide> filteredRides = rides
				// filter out rides that do not start or stop in NYC
				.filter(new NYCFilter());

		// print the filtered stream
		printOrTest(filteredRides);

		// run the cleansing pipeline
		env.execute("Taxi Ride Cleansing");
	}

	private static class NYCFilter implements FilterFunction<TaxiRide> {

		@Override
		public boolean filter(TaxiRide taxiRide) throws Exception {
			return GeoUtils.isInNYC(taxiRide.startLon, taxiRide.startLat)
					&& GeoUtils.isInNYC(taxiRide.endLon, taxiRide.endLat);
		}
	}

	}


## Test_RideCleansingExercise

![Test_RideCleansingExercise](https://github.com/dimpls/BigData/blob/main/lab3/images/test_RideCleanisingExercise.png)

## RidesAndFaresExercise

	public class RidesAndFaresExercise extends ExerciseBase {
	public static void main(String[] args) throws Exception {

		ParameterTool params = ParameterTool.fromArgs(args);
		final String ridesFile = params.get("rides", pathToRideData);
		final String faresFile = params.get("fares", pathToFareData);

		final int delay = 60;					// at most 60 seconds of delay
		final int servingSpeedFactor = 1800; 	// 30 minutes worth of events are served every second

		// set up streaming execution environment
		StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		env.setParallelism(ExerciseBase.parallelism);

		DataStream<TaxiRide> rides = env
				.addSource(rideSourceOrTest(new TaxiRideSource(ridesFile, delay, servingSpeedFactor)))
				.filter((TaxiRide ride) -> ride.isStart)
				.keyBy("rideId");

		DataStream<TaxiFare> fares = env
				.addSource(fareSourceOrTest(new TaxiFareSource(faresFile, delay, servingSpeedFactor)))
				.keyBy("rideId");

		DataStream<Tuple2<TaxiRide, TaxiFare>> enrichedRides = rides
				.connect(fares)
				.flatMap(new EnrichmentFunction());

		printOrTest(enrichedRides);

		env.execute("Join Rides with Fares (java RichCoFlatMap)");
	}

	public static class EnrichmentFunction extends RichCoFlatMapFunction<TaxiRide, TaxiFare, Tuple2<TaxiRide, TaxiFare>> {

		private ValueState<TaxiRide> rideState;
		private ValueState<TaxiFare> fareState;

		@Override
		public void open(Configuration config) throws Exception {
			ValueStateDescriptor<TaxiRide> taxiRideDescriptor =
					new ValueStateDescriptor<>("storedTaxiRide", TaxiRide.class);
			rideState = getRuntimeContext().getState(taxiRideDescriptor);

			ValueStateDescriptor<TaxiFare> taxiFareDescriptor =
					new ValueStateDescriptor<>("storedTaxiFare", TaxiFare.class);
			fareState = getRuntimeContext().getState(taxiFareDescriptor);
		}

		@Override
		public void flatMap1(TaxiRide ride, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {
			TaxiFare fare = fareState.value();
			if (fare != null) {
				Tuple2<TaxiRide, TaxiFare> rideAndFare = new Tuple2<>(ride, fare);

				out.collect(rideAndFare);

				fareState.clear();
			}
		}

		@Override
		public void flatMap2(TaxiFare fare, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {
			TaxiRide ride = rideState.value();
			if (ride != null) {
				Tuple2<TaxiRide, TaxiFare> rideAndFare = new Tuple2<>(ride, fare);

				out.collect(rideAndFare);

				rideState.clear();
			} else {
				fareState.update(fare);
			}
		}
	}
	}

## Test_RidesAndFaresExercise
![Test_RidesAndFaresExercise](https://github.com/dimpls/BigData/blob/main/lab3/images/test_RidesAndFaresExercise.png)

## HourlyTipsExerxise
	public class HourlyTipsExercise extends ExerciseBase {

	public static void main(String[] args) throws Exception {

		// read parameters
		ParameterTool params = ParameterTool.fromArgs(args);
		final String input = params.get("input", ExerciseBase.pathToFareData);

		final int maxEventDelay = 60;       // events are out of order by max 60 seconds
		final int servingSpeedFactor = 600; // events of 10 minutes are served in 1 second

		// set up streaming execution environment
		StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
		env.setParallelism(ExerciseBase.parallelism);

		// start the data generator
		DataStream<TaxiFare> fares = env.addSource(fareSourceOrTest(new TaxiFareSource(input, maxEventDelay, servingSpeedFactor)));

		SingleOutputStreamOperator<Tuple3<Long, Long, Float>> output = fares
				.keyBy((TaxiFare f) -> f.driverId)
				.timeWindow(Time.hours(1))
				.process(new TipAccumulator())
				.timeWindowAll(Time.hours(1))
				.maxBy(2);

		printOrTest(output);
		System.out.println(env.getExecutionPlan());
		env.execute("Hourly Tips (java)");
	}

	public static class TipAccumulator extends ProcessWindowFunction<
			TaxiFare, Tuple3<Long, Long, Float>, Long, TimeWindow> {
		@Override
		public void process(Long key, Context context, Iterable<TaxiFare> fares, Collector<Tuple3<Long, Long, Float>> out) throws Exception {
			float sumOfTips = StreamSupport.stream(fares.spliterator(), false)
					.map(f -> f.tip)
					.reduce(0F, Float::sum);

			out.collect(new Tuple3<>(context.window().getEnd(), key, sumOfTips));

		}
	}
	}
## Test_HourlyTipsExerxise
![Test_RidesAndFaresExercise](https://github.com/dimpls/BigData/blob/main/lab3/images/test_HourlyTipsExerxise.png)

## ExpiringStateExercise

	public class ExpiringStateExercise extends ExerciseBase {
	static final OutputTag<TaxiRide> unmatchedRides = new OutputTag<TaxiRide>("unmatchedRides") {};
	static final OutputTag<TaxiFare> unmatchedFares = new OutputTag<TaxiFare>("unmatchedFares") {};

	public static void main(String[] args) throws Exception {

		ParameterTool params = ParameterTool.fromArgs(args);
		final String ridesFile = params.get("rides", ExerciseBase.pathToRideData);
		final String faresFile = params.get("fares", ExerciseBase.pathToFareData);

		final int maxEventDelay = 60;           // events are out of order by max 60 seconds
		final int servingSpeedFactor = 600; 	// 10 minutes worth of events are served every second

		// set up streaming execution environment
		StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
		env.setParallelism(ExerciseBase.parallelism);

		DataStream<TaxiRide> rides = env
				.addSource(rideSourceOrTest(new TaxiRideSource(ridesFile, maxEventDelay, servingSpeedFactor)))
				.filter((TaxiRide ride) -> (ride.isStart && (ride.rideId % 1000 != 0)))
				.keyBy(ride -> ride.rideId);

		DataStream<TaxiFare> fares = env
				.addSource(fareSourceOrTest(new TaxiFareSource(faresFile, maxEventDelay, servingSpeedFactor)))
				.keyBy(fare -> fare.rideId);

		SingleOutputStreamOperator processed = rides
				.connect(fares)
				.process(new EnrichmentFunction());

		printOrTest(processed.getSideOutput(unmatchedFares));

		env.execute("ExpiringStateExercise (java)");
	}

	public static class EnrichmentFunction extends KeyedCoProcessFunction<Long, TaxiRide, TaxiFare, Tuple2<TaxiRide, TaxiFare>> {

		private ValueState<TaxiRide> rideState;
		private ValueState<TaxiFare> fareState;

		@Override
		public void open(Configuration config) throws Exception {
			ValueStateDescriptor<TaxiRide> taxiRideDescriptor =
					new ValueStateDescriptor<>("storedTaxiRide", TaxiRide.class);
			rideState = getRuntimeContext().getState(taxiRideDescriptor);

			ValueStateDescriptor<TaxiFare> taxiFareDescriptor =
					new ValueStateDescriptor<>("storedTaxiFare", TaxiFare.class);
			fareState = getRuntimeContext().getState(taxiFareDescriptor);
		}

		@Override
		public void onTimer(long timestamp, OnTimerContext context, Collector<Tuple2<TaxiRide, TaxiFare>> collector) throws Exception {

			if (rideState.value() != null) {
				context.output(unmatchedRides, rideState.value());
				rideState.clear();
			}

			if (fareState.value() != null) {
				context.output(unmatchedFares, fareState.value());
				fareState.clear();
			}
		}

		@Override
		public void processElement1(TaxiRide ride, Context context, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {

			TaxiFare fare = fareState.value();

			if (fare != null) {
				fareState.clear();
				context.timerService().deleteEventTimeTimer(fare.getEventTime());
				out.collect(new Tuple2(ride, fare));
			} else {
				rideState.update(ride);
				context.timerService().registerEventTimeTimer(ride.getEventTime());
			}
		}

		@Override
		public void processElement2(TaxiFare fare, Context context, Collector<Tuple2<TaxiRide, TaxiFare>> out) throws Exception {

			TaxiRide ride = rideState.value();

			if (ride != null) {
				rideState.clear();
				context.timerService().deleteEventTimeTimer(ride.getEventTime());
				out.collect(new Tuple2(ride, fare));
			} else {
				fareState.update(fare);
				context.timerService().registerEventTimeTimer(fare.getEventTime());
			}
		}
	}
	}
## Test_ExpiringStateExercise

![Test_RidesAndFaresExercise](https://github.com/dimpls/BigData/blob/main/lab3/images/test_ExpiringStateExercise.png)

