# Airbnb_listings_NYC

## SQL queries

## Import CSV into MySQL through LOAD DATA INFILE Statement
    LOAD DATA INFILE 'C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/airbnb_listings_NYC.csv'

    INTO TABLE airbnb_listings

    FIELDS TERMINATED BY ',' 

    ENCLOSED BY '"'

    LINES TERMINATED BY '\n'

    IGNORE 1 LINES

    (@id, @name, @host_id, @host_name, @neighbourhood_group, @neighbourhood, @latitude, @longitude, @room_type, @price, @minimum_nights, @number_of_reviews, @last_review, @reviews_per_month, 
    @calculated_host_listings_count, @availability_365)

    SET 

    id = NULLIF(@id, ''),
    
    name = NULLIF(@name, ''),
    
    host_id = NULLIF(@host_id, ''),
    
    host_name = NULLIF(@host_name, ''),
    
    neighbourhood_group = NULLIF(@neighbourhood_group, ''),
    
    neighbourhood = NULLIF(@neighbourhood, ''),
    
    latitude = NULLIF(@latitude, ''),
    
    longitude = NULLIF(@longitude, ''),
    
    room_type = NULLIF(@room_type, ''),
    
    price = NULLIF(@price, ''),
    
    minimum_nights = NULLIF(@minimum_nights, ''),
    
    number_of_reviews = NULLIF(@number_of_reviews, ''),
    
    last_review = CASE
    
        WHEN @last_review = '' THEN NULL
        
        WHEN @last_review REGEXP '^[0-9]{2}-[0-9]{2}-[0-9]{4}$' THEN STR_TO_DATE(@last_review, '%d-%m-%Y')
        
        WHEN @last_review REGEXP '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' THEN STR_TO_DATE(@last_review, '%Y-%m-%d')
        
        ELSE NULL
        
    END,
    
    reviews_per_month = NULLIF(@reviews_per_month, ''),
    
    calculated_host_listings_count = NULLIF(@calculated_host_listings_count, ''),
    
    availability_365 = NULLIF(@availability_365, '');

  ## Query to remove duplicates

    DELETE FROM airbnb_listings
    
    WHERE id IN (

    SELECT id FROM (
    
        SELECT 
        
            id,
            
            ROW_NUMBER() OVER (PARTITION BY id, name, host_id, host_name, neighbourhood_group, neighbourhood, latitude, longitude, room_type, price, minimum_nights, number_of_reviews, last_review, 
            reviews_per_month, calculated_host_listings_count, availability_365 ORDER BY id) AS row_num
            
        FROM 
        
            airbnb_listings
            
    ) AS subquery
    
    WHERE row_num > 1
    
    );

## Query to delete host names with null values

    DELETE FROM airbnb_listings 

    WHERE

    host_name IS NULL;

## Query for Imputation and Flagging

     -- Impute 'last_review' with a placeholder date
     UPDATE airbnb_listings
     SET last_review = '2000-01-01'
     WHERE last_review IS NULL;

     -- Calculate the mean of 'reviews_per_month'. Create a temporary table to store the mean value
     CREATE TEMPORARY TABLE mean_reviews AS
     SELECT AVG(reviews_per_month) AS mean_reviews_per_month
     FROM airbnb_listings
     WHERE reviews_per_month IS NOT NULL;

     -- impute 'reviews_per_month' with the mean value
     UPDATE airbnb_listings
     SET reviews_per_month = (SELECT mean_reviews_per_month FROM mean_reviews)
     WHERE reviews_per_month IS NULL;

     -- Create flag columns for 'last_review' and 'reviews_per_month'
     ALTER TABLE airbnb_listings
     ADD COLUMN last_review_null INT DEFAULT 0,
     ADD COLUMN reviews_per_month_null INT DEFAULT 0;

    -- Update flag columns based on imputed values
    UPDATE airbnb_listings
    SET last_review_null = 1
    WHERE last_review = '2000-01-01';

    UPDATE airbnb_listings
    SET reviews_per_month_null = 1
    WHERE reviews_per_month = (SELECT mean_reviews_per_month FROM mean_reviews);

## Host Information Analysis

    SELECT 
          host_name, 
          COUNT(id) AS listings_count, 
          ROUND(AVG(number_of_reviews),2) AS avg_reviews, 
          ROUND(AVG(reviews_per_month),2) AS avg_reviews_per_month
    FROM 
        airbnb_listings
    GROUP BY 
        host_name
    ORDER BY 
        COUNT(*) DESC;

## Neighborhood Information Analysis

     SELECT 
           neighbourhood_group, 
           neighbourhood, 
           COUNT(id) AS listings_count, 
           AVG(price) AS avg_price, 
           AVG(number_of_reviews) AS avg_reviews
    FROM 
           airbnb_listings
    GROUP BY 
           neighbourhood_group, 
           neighbourhood
    ORDER BY 
           COUNT(*) DESC, neighbourhood_group DESC;

## Room distribution

    SELECT
        room_type,
        COUNT(*) AS room_count
    FROM
        airbnb_listings
    GROUP BY
        room_type
    ORDER BY
        room_count DESC;

## Busiest hosts

    SELECT
         host_name,
         AVG(reviews_per_month) AS avg_reviews_per_month
    FROM
         airbnb_listings
    GROUP BY
         host_name
    ORDER BY 
         avg_reviews_per_month DESC;

## Highest traffic neighborhood

    SELECT
         neighbourhood,
         AVG(reviews_per_month) AS avg_reviews
    FROM
        airbnb_listings
    GROUP BY
        neighbourhood
    ORDER BY 
        avg_reviews DESC;

