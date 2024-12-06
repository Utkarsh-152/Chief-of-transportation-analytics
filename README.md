# Chief-of-transportation-analytics

## SQL Queris for some AD-Hoc questions 

1. City Level Fare and Trip Summary Report
    ```sql
      SELECT
      		c.city_name,
      		COUNT(t.trip_id) AS total_trips,
      		AVG(t.fare_amount / t.distance_travelled_km) AS avg_fare_per_km,
      		AVG(t.fare_amount) AS avg_fare_per_trip,
      		(COUNT(t.trip_id) / (SELECT COUNT(*) FROM fact_trips)) * 100 AS percent_contribution_to_total_trips
      	FROM
          trips_db.fact_trips t
      INNER JOIN dim_city c ON t.city_id = c.city_id
      GROUP BY
          c.city_name;

2. Monthly City Level Trips Target Performance Repert
    ```sql    
      SELECT
        c.city_name,
        d.month_name,
        COUNT(t.trip_id) AS actual_trips,
        tt.total_target_trips AS target_trips,
        CASE
            WHEN COUNT(t.trip_id) > tt.total_target_trips THEN 'Above Target'
            ELSE 'Below Target'
        END AS performance_status,
        (COUNT(t.trip_id) - tt.total_target_trips) / tt.total_target_trips * 100 AS percent_difference
      FROM
        trips_db.fact_trips t
      INNER JOIN trips_db.dim_city c ON t.city_id = c.city_id
      INNER JOIN trips_db.dim_date d ON t.date = d.date
      INNER JOIN targets_db.monthly_target_trips tt ON t.city_id = tt.city_id AND d.start_of_month = tt.month
      GROUP BY
        c.city_name, d.month_name, tt.total_target_trips;

3. City Level Repeat Passenger Trip Frequency Report
    ```sql
      WITH RepeatPassengerCounts AS (
      SELECT
          city_id,
          SUM(CASE WHEN trip_count = '2-Trips' THEN repeat_passenger_count ELSE 0 END) AS two_trips,
          SUM(CASE WHEN trip_count = '3-Trips' THEN repeat_passenger_count ELSE 0 END) AS three_trips,
          SUM(CASE WHEN trip_count = '4-Trips' THEN repeat_passenger_count ELSE 0 END) AS four_trips,
          SUM(CASE WHEN trip_count = '5-Trips' THEN repeat_passenger_count ELSE 0 END) AS five_trips,
          SUM(CASE WHEN trip_count = '6-Trips' THEN repeat_passenger_count ELSE 0 END) AS six_trips,
          SUM(CASE WHEN trip_count = '7-Trips' THEN repeat_passenger_count ELSE 0 END) AS seven_trips,
          SUM(CASE WHEN trip_count = '8-Trips' THEN repeat_passenger_count ELSE 0 END) AS eight_trips,
          SUM(CASE WHEN trip_count = '9-Trips' THEN repeat_passenger_count ELSE 0 END) AS nine_trips,
          SUM(CASE WHEN trip_count = '10-Trips' THEN repeat_passenger_count ELSE 0 END) AS ten_trips,
          SUM(repeat_passenger_count) AS total_repeat_passengers
      FROM
          trips_db.dim_repeat_trip_distribution
      GROUP BY
          city_id
      )
      SELECT
          c.city_name,
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.two_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "2-Trips",
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.three_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "3-Trips",
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.four_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "4-Trips",
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.five_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "5-Trips",
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.six_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "6-Trips",
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.seven_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "7-Trips",
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.eight_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "8-Trips",
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.nine_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "9-Trips",
          CASE WHEN r.total_repeat_passengers > 0 THEN (r.ten_trips * 100.0 / r.total_repeat_passengers) ELSE 0 END AS "10-Trips"
      FROM
          trips_db.dim_city c
      INNER JOIN RepeatPassengerCounts r ON c.city_id = r.city_id;

4. Identify Cities With Highest and Lowest Total New Passengers
    ```sql
      WITH TotalNewPassengers AS (
      SELECT
          c.city_id,
          c.city_name,
          SUM(f.new_passengers) AS total_new_passengers
      FROM
          trips_db.dim_city c
      INNER JOIN trips_db.fact_passenger_summary f ON c.city_id = f.city_id
      GROUP BY c.city_id, c.city_name
      ),
      RankedCities AS (
          SELECT
              city_name,
              total_new_passengers,
              ROW_NUMBER() OVER (ORDER BY total_new_passengers DESC) AS rank_highest,
              ROW_NUMBER() OVER (ORDER BY total_new_passengers ASC) AS rank_lowest
          FROM TotalNewPassengers
      )
      SELECT
          city_name,
          total_new_passengers,
          CASE
              WHEN rank_highest <= 3 THEN 'Top 3'
              WHEN rank_lowest <= 3 THEN 'Bottom 3'
              ELSE NULL
          END AS city_category
      FROM RankedCities
      WHERE rank_highest <= 3 OR rank_lowest <= 3;

5. Identify Month With Highest Revenue For Each City
    ```sql
      WITH CityMonthlyRevenue AS (
      SELECT
          f.city_id,
          c.city_name,
          d.month_name,
          SUM(f.fare_amount) AS revenue
      FROM
          trips_db.fact_trips f
      INNER JOIN trips_db.dim_city c ON f.city_id = c.city_id
      INNER JOIN trips_db.dim_date d ON f.date = d.date
      GROUP BY f.city_id, c.city_name, d.month_name
      ),
      CityTotalRevenue AS (
          SELECT
              city_id,
              SUM(revenue) AS total_revenue
          FROM CityMonthlyRevenue
          GROUP BY city_id
      ),
      MaxRevenueMonth AS (
          SELECT
              r.city_name,
              r.month_name AS highest_revenue_month,
              r.revenue,
              t.total_revenue,
              (r.revenue * 100.0 / t.total_revenue) AS percentage_contribution
          FROM CityMonthlyRevenue r
          INNER JOIN CityTotalRevenue t ON r.city_id = t.city_id
          WHERE r.revenue = (
              SELECT MAX(revenue)
              FROM CityMonthlyRevenue r2
              WHERE r2.city_id = r.city_id
          )
      )
      SELECT
          city_name,
          highest_revenue_month,
          revenue,
          percentage_contribution
      FROM MaxRevenueMonth;



6. Repeat Passenger Rate Analysis
    ```sql
      WITH MonthlyRepeatPassengerRate AS (
      SELECT
          c.city_id,
          c.city_name,
          d.month_name,
          ps.total_passengers,
          ps.repeat_passengers,
          (ps.repeat_passengers * 100.0 / ps.total_passengers) AS monthly_repeat_passenger_rate
      FROM
          trips_db.fact_passenger_summary ps
      INNER JOIN trips_db.dim_city c ON ps.city_id = c.city_id
      INNER JOIN trips_db.dim_date d ON ps.month = d.start_of_month
      ),
      CityRepeatPassengerRate AS (
          SELECT
              c.city_id,
              c.city_name,
              SUM(ps.repeat_passengers) AS total_city_repeat_passengers,
              SUM(ps.total_passengers) AS total_city_passengers,
              (SUM(ps.repeat_passengers) * 100.0 / SUM(ps.total_passengers)) AS city_repeat_passenger_rate
          FROM
              trips_db.fact_passenger_summary ps
          INNER JOIN trips_db.dim_city c ON ps.city_id = c.city_id
          GROUP BY c.city_id, c.city_name
      )
      SELECT
          m.city_name,
          m.month_name,
          m.total_passengers,
          m.repeat_passengers,
          m.monthly_repeat_passenger_rate,
          crr.city_repeat_passenger_rate
      FROM
          MonthlyRepeatPassengerRate m
      INNER JOIN
          CityRepeatPassengerRate crr ON m.city_id = crr.city_id
      GROUP BY
          m.city_name, m.month_name, m.total_passengers, m.repeat_passengers, m.monthly_repeat_passenger_rate, crr.city_repeat_passenger_rate;

