# Notes on CppCon 2016: Howard Hinnant “A ＜chrono＞ Tutorial"

Reference:
<https://www.youtube.com/watch?v=P32hvk8b13M&ab_channel=CppCon>

## Background and motivation

- header *<`chrono`>*, everything is in *namespace std::chrono*
- *strong typed* and avoided ambiguity

## Time Duration

- hours
- minutes
- seconds
- milliseconds
- microseconds
- nanoseconds

## Seconds

- *std::chrono::seconds*
- a class, wrapper of *int64_t* or *long long*, sizeof(seconds) == 8
- Scalar-like construction
  - seconds s; // no initialization
  - seconds s{}; // zero initialization
  - seconds s = 3; // error: Not implicitly constructible from int
  - seconds s{3}; // ok: 3 seconds
- Print-out
  - cout << s << '\n'; // unfortunately, not ok (could be seen as a bug)
  - cout << s.count() << "s\n"; // ok
- **Important**: Conversion from int or long long to seconds is not allowed, since it would trigger type ambiguity and is contradict to the spirit and objective of <`chrono`>!
                 For example, seconds s = 3; what is 3? 3s? 3ms? 3minutes? or even 3 hours?
- zero overhead in arithmetic operation, when compared with operation on int64_t
- range: +/- 292 B years, can queried by seconds::min() and seconds::max()

## milliseconds

- milliseconds works just like seconds, except with range of only +/- 292 M years.
- **Important** do not manually convert values between different time duration, <`chrono`> can handle and let <`chrono`> handle all the conversions. Letting <`chrono`> handling the conversion leads to only two results, either compiling and working or failed to compiled. Do not try to escape the compilation failure through manually converting the value derived from .count().
- The typical reason for using count() should be I/O or interfacing with legacy code.  

## Loss during conversion

- loss-less conversion: e.g., from seconds to milliseconds
- non-loss-less conversion: e.g., from milliseconds to seconds
  - not compile until given special syntax
    - seconds x = 3400ms; // error: no conversion
    - seconds x = duration_cast<`seconds`>(3400ms); // ms
    - "duration_cast" means conversion with truncation towards zero
    - truncation towards zeros means down-truncation for positive values and up-truncation for negative values
    - Other truncating strategies, afte C++17), includes floor<`duration`>, ceil<`duration`> and round<`duration`>

## Generalized representation

- for seconds
  - "duration<`rep`>" to represent seconds a ref type (ref should be a arithmetic type)
  - e.g., using fseconds = duration<`float`>
    - will allow implict conversion without loss
    - free from using duration_cast
- for other units
  - via template <`class T`>
  
  - e.g., ```c++
          *using my_ms = std::chrono::duration<T, std::milli>;
          * my_ms<float> t;*```

- The standard specifies
  - using nanoseconds = duration<int_least64_t, nano>;
  - using microseconds = duration<int_least55_t, micro>;
  - using milliseconds = duration<int_least45_t, milli>;
  - using seconds = duration<int_least35_t, ratio<1>>;
  - using minutes = duration<int_least29_t, ratio<60>>;
  - using hours = duration<int_least23_t, ratio<3600>>;

- Durations is actually a template

  - ```c++
    template<class Rep, class Period = ratio<1>>
    class duration{
     public:
      using rep = Rep;
      using period = Period;
      // ...
    }
    ```

  - e.g., milliseconds::rep is int64_t;
          milliseconds::period::num is 1;
          milliseconds::period::den is 1000;
  - custom duration is possible
    - e.g., *using frames = duration<int32_t, ratio<1,60>>*;
    - e.g., *using days = duration <int, ratio_multiply<ratio<24>, hours::period>>*
    - e.g., *using days = duration <int, ratio<86400>>* (same to above declaration)
  - Calculation and conversion between different periods are done efficiently at compile time

## Summary for time duration

- seconds is just a wrapper of long long implemented as *duration<int64_t, ratio<1,1>> most of the time*
- use the weakest type conversion possible
- Implicit if at all possible
    duration_cast if you need to specify truncation

## Time point

- A time_point refers to a specific point in time, w.r.t. some clock and has a precision of some duration

- ``` c++
  template <class Clock, class Duration = typename Clock::duration>
  class time_point {
    Duration d_;
   public:
    using clock = Clock;
    using duration = Duration;
    // ...
  }
  ```

- time_points and durations can have the exact same representation, but they mean different things.
- arithmetic operation of time_point (tp) involve durations (d)
  - auto d = tp1 - tp2;
  - auto tp2 = tp1 + d;
  - These algebra are 100% self consistent and type-checked at compile time.
- time_points convert much like the durations do: implicitly when the conversion is loss-less
- to force a truncation error, use *time_point_cast<'rep'>(tp)*
- to force a time_point to duration conversion, use *.time_since_epoch()*

## Clocks

- A clock is a bundle of a duration, a time_point and a static function to get the current time
- Everey time_point is associated with a clock
- time_points assoiciated with different clocks do not convert to one another
- Two useful std-supplied clocks:
  - std::chrono::system_clock
    - use system_clock when you need time_points that must relate to some calendar
  - std::chrono::steady_clock
    - steady_clock is like a stopwatch and good for timing, but cannot gives time of the day  
  - std::chrono::high_resolution_clock (ignore this one, since it's usually alias for one of the above two)
- getting time_points can be like
  - clock::time_point tp = clock::now();
  - auto tp = clock::now();
- Time since epoch (de facto standard 1970-01-01 00:00:00 UTC)

```c++
  auto tp = time_point_cast<second>(system_clock::now());
  cout<< tp.time_since_epoch().count() << "s\n";
  // 1469456123s
  auto td = time_point_cast<days>(tp);
  cout << td.time_since_epoch().count() << " days/n";
  // 17007 days
```

## Summary: <`chrono`>

- Based on three modules
  - duration
  - time_point
  - clock

- Compile-time errors are favored over run-time errors.
- As efficient as hand-written code (or better)
- Feature rich, but you don't pay for features you don't use
- Not a bleeding edge code, but a best practice
