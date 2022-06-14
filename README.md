# SQL (MySQL workbench + Command client, SQL Server, Oracle)

Example Database SAKILA 

SUBQUERIES 

SELECT customer_id, first_name, last_name
FROM sakila.customer
WHERE customer_id = (SELECT MAX(customer_id) FROM sakila.customer);

SELECT customer_id cust_id, first_name, last_name
FROM sakila.customer
WHERE customer_id = MAX(customer_id) #These built in functions work on Select statements only not where clause
#So you would need a subquery to find the max cust _id and this cust_id first name , last name

	#Self contained subqueries called: NONCORRELATED SUBQUERIES
	#with inequality condition
    SELECT city_id, city
    FROM sakila.city
    WHERE country_id <>
		(SELECT country_id FROM sakila.country WHERE country = 'India');
	
    #Can i use a join to work with both tables instead of using a subquery? A: YES but its more 
    #lines of query
	SELECT *
    FROM sakila.city;
    SELECT *
    FROM sakila.country
    
    #A: yes 
    SELECT city_id, city
    FROM sakila.city c
		INNER JOIN sakila.country co
		ON c.country_id = co.country_id 
	WHERE co.country <> 'India'
    
    #cant have a subquery with multiple rows if its an equality condition 
    SELECT city_id, city
    FROM sakila.city
	WHERE country_id <>
		(SELECT country_id FROM sakila.country WHERE country <> 'India');
        #ERROR: returns more than 1 row
        
	#Multi-row, single column subqueries (IN AND NOT IN operators allow multiple row subqueries)
    SELECT city_id, city
    FROM sakila.city
    WHERE country_id IN 
		(SELECT country_id 
         FROM sakila.country 
         WHERE country IN ('Mexico', 'Canada'));
         
	SELECT city_id, city
    FROM sakila.city
    WHERE country_id NOT IN 
		(SELECT country_id 
         FROM sakila.country 
         WHERE country IN ('Mexico', 'Canada'));
	
    #The ALL operator allows the equality condition to take in more than one value
    SELECT first_name, last_name
    FROM sakila.customer
    WHERE customer_id <> ALL 
		(SELECT customer_id
         FROM sakila.payment
         WHERE amount = 0);
         
		#can i not use an IN (it seems like i can but NOT IN doesnt check  against NULL values)
	SELECT first_name, last_name
    FROM sakila.customer
    WHERE customer_id NOT IN 
		(SELECT customer_id
         FROM sakila.payment
         WHERE amount = 0);
    
    #Another example using the ALL operator but in the HAVING clause. The ALL & ANY operators are 
    # only used with HAVING AND WHERE clause
    SELECT customer_id, count(*)
    FROM sakila.rental
    GROUP BY customer_id
    HAVING count(*) > ALL
		(SELECT count(*)
         FROM sakila.rental r
			INNER JOIN sakila.customer c
            ON r.customer_id=c.customer_id 
            INNER JOIN sakila.address a
            ON c.address_id = a.address_id
            INNER JOIN sakila.city ct
            ON a.city_id = ct.city_id
            INNER JOIN sakila.country co
            ON ct.country_id= co.country_id
		WHERE co.country IN ('United States', 'Mexico', 'Canada')
        GROUP BY r.customer_id
        );
	#The ANY operator. Like the ALL operator, but unlike ALL, ANY evaluates to true as soon
    # as a single comparison is favorable. Not sure what is the difference
    SELECT customer_id, sum(amount)
	FROM sakila.payment
    GROUP BY customer_id
    HAVING sum(amount) > ANY
		(SELECT sum(p.amount)
         FROM sakila.payment p
			INNER JOIN sakila.customer c
            ON p.customer_id = c.customer_id
            INNER JOIN sakila.address a
            ON c.address_id = a.address_id
            INNER JOIN sakila.city ct
            ON a.city_id = ct.city_id
            INNER JOIN sakila.country co
            ON ct.country_id = co.country_id
		WHERE co.country IN ('Bolivia', 'Paraguay', 'Chile')
        GROUP BY co.country
        );

	#MULTICOLUMN SUBQUERIES
	SELECT fa.actor_id, fa.film_id
	FROM sakila.film_actor fa
	WHERE fa.actor_id IN
		(SELECT actor_id FROM sakila.actor WHERE last_name = 'MONROE')
		AND fa.film_id IN
		(SELECT film_id FROM sakila.film WHERE rating= 'PG');

	#CROSS JOIN 
	SELECT actor_id, film_id
	FROM sakila.film_actor
	WHERE (actor_id, film_id) IN 
		(SELECT a.actor_id, f.film_id
		FROM sakila.actor a
			CROSS JOIN sakila.film f
		WHERE a.last_name = 'MONROE'
		AND f.rating = 'PG');

	#CORRELATED SUBQUERIES 
    #(subqueries that are dependent on their containing statement)
    SELECT c.first_name, c.last_name
    FROM sakila.customer c
    WHERE 20 = 
		(SELECT count(*) FROM sakila.rental r
		 WHERE r.customer_id = c.customer_id);
	 #correlated subqueries can have performance issues if containing subquery returns a large number
     #in this case the subquery runs every time as compared to the containing query. 
     #The containing query ran once and collected the 599 rows, then the whole correlated subquery 
     #runs once every 599 rows. 
     #You can use equality conditions, ranges, with correlated subqueries
     
     SELECT c.first_name, c.last_name
     FROM sakila.customer c
     WHERE 
		(SELECT sum(p.amount) FROM sakila.payment p
		 WHERE p.customer_id = c.customer_id)
         BETWEEN 180 AND 240;
         
	#The EXISTS OPERATOR
    #the most common operator used to build conditions that utilize correlated subqueries
    #@you use the EXISTS operator when you want to identify that a relationship exists without 
    #regard for the quantity
    #The exists operator doesnt care what data is returned from the subquery 
		  SELECT c.first_name, c.last_name
          FROM sakila.customer c
          WHERE EXISTS
            (SELECT 1 from sakila.rental r
			 WHERE r.customer_id = c.customer_id
               AND date(r.rental_date) < '2005-05-25');
               
		#NOT EXISTS 
        #This query helps us find instances where subqueries return no rows
        SELECT a.first_name, a.last_name
        FROM sakila.actor a
        WHERE NOT EXISTS 
          (SELECT 1 
           FROM sakila.film_actor fa
		      INNER JOIN sakila.film f ON f.film_id = fa.film_id
		   WHERE fa.actor_id = a.actor_id
             AND f.rating = 'R');
		#the UPDATE, DELETE, and INSERT statements also rely heavily on subqueries too
        UPDATE sakila.customer c
        SET c.last_name = 
			(SELECT max(r.rental_date) FROM sakila.rental r
             WHERE r.customer_id = c.customer_id);   #problem with this is that if there is a
             #customer with no film rental then rental date will be set to null 
             #using the EXISTS op[erator protects the data. 
		UPDATE sakila.customer c
        SET c.last_name = 
			(SELECT max(r.rental_date) FROM sakila.rental r
             WHERE r.customer_id = c.customer_id)
		 WHERE EXISTS #set only executes if the where statement evaluates to TRUE
			(SELECT 1 FROM sakila.rental r
             WHERE r.customer_id = c.customer_id)
             #THE SET clause only executes if the UPDATE condition WHERE evaluates to true
             #meaning there was at least one rental found for the customer, preventing the 
             #last_update column from being overwritten with a null.
             
             #the DELETE statement also uses correlated subqueries
             DELETE FROM sakila.customer
             WHERE 365 < ALL 
               (SELECT datediff(now(), r.rental_date) days_since_last_rental
                 FROM sakila.rental r
                 WHERE r.customer_id = customer.customer_id);
                 
		#When to use Subqueries
        SELECT c.first_name, c.last_name,
			pymnt.num_rentals, pymnt.tot_payments
		FROM sakila.customer c 
			INNER JOIN 
             (SELECT customer_id, 
				count(*) num_rentals, sum(amount) tot_payments
			  FROM sakila.payment
              GROUP BY customer_id
              ) pymnt
              ON c.customer_id = pymnt.customer_id;
			#subqueries in the FROM clause must be noncorrelated 
	#Data Fabrication
    SELECT pymnt_grps.name, count(*) num_customers
    FROM
    (SELECT customer_id,
      count(*) num_rentals, sum(amount) tot_payments
	 FROM sakila.payment
     GROUP BY customer_id
     ) pymnt
     INNER JOIN
	    (SELECT 'Small fry' name, 0 low_limit, 74.99 high_limit
        UNION ALL
        SELECT 'Average Joes' name, 75 low_limit, 149.99 high_limit
        UNION ALL
        SELECT 'Heavy Hitters' name, 150 low_limit, 999999.99 high_limit
		) pymnt_grps
        ON pymnt.tot_payments
          BETWEEN pymnt_grps.low_limit AND pymnt_grps.high_limit
	GROUP BY pymnt_grps.name;
	
    #Task oriented subqueries 
     SELECT first_name, last_name, city, sum(amount) tot_payments, count(*) tot_rentals
     FROM sakila.payment p
       INNER JOIN sakila.customer c
       ON p.customer_id = c.customer_id 
       INNER JOIN sakila.address a 
       ON c.address_id = a.address_id
       INNER JOIN sakila.city ct
       ON a.city_id = ct.city_id
	GROUP BY c.first_name, c.last_name, ct.city;
	
    SELECT c.first_name, c.last_name, 
    ct.city,
    pymnt.tot_payments, pymnt.tot_rentals
    FROM 
     (SELECT customer_id,
		 count(*) tot_rentals, sum(amount) tot_payments
	  FROM sakila.payment
      GROUP BY customer_id  #seems like multiple groupings might take longer to run while this one only does one
      ) pymnt
      INNER JOIN sakila.customer c
      ON pymnt.customer_id = c.customer_id
      INNER JOIN sakila.address a
      ON c.address_id = a.address_id
      INNER JOIN sakila.city ct
      ON a.city_id = ct.city_id
	  
	#Common table expressions (CTE)
    #Appears at the top in a WITH clause, can have multiple CTES, and make code more readable. 
    #allows CTE's to refer to  any other CTE defined above it in the same with clause.
    #here the second CTE refers to the first and the third refers to the second
    WITH actors_s AS
    (SELECT actor_id, first_name, last_name
     FROM sakila.actor
	 WHERE last_name LIKE 'S%'
     ),
     actors_s_pg AS
     (SELECT s.actor_id, s.first_name, s.last_name,
        f.film_id, f.title
	  FROM actors_s s
		INNER JOIN sakila.film_actor fa
        ON s.actor_id = fa.actor_id
        INNER JOIN sakila.film f
        ON f.film_id = fa.film_id
	  WHERE f.rating = 'PG'
      ),
      actors_s_pg_revenue AS
      (SELECT spg.first_name, spg.last_name, p.amount
       FROM actors_s_pg spg
		  INNER JOIN sakila.inventory i
          ON i.film_id = spg.film_id
          INNER JOIN sakila.rental r
          ON i.inventory_id = r.inventory_id
          INNER JOIN sakila.payment p 
          ON r.rental_id = p.rental_id
	 ) -- end of with class
     SELECT spg_rev.first_name, spg_rev.last_name,
     sum(spg_rev.amount) tot_revenue
     FROM actors_s_pg_revenue spg_rev
     GROUP BY spg_rev.first_name, spg_rev.last_name
     ORDER BY 3 DESC;
     
     #SUBQUERIES as Expression Generators
     #single-column, single-row scalar subqueries
     #can be used anywhere an expression can appear like SELECT, ORDER BY, and the VALUES of an INSERT statement
     SELECT 
      (SELECT c.first_name FROM sakila.customer c #1
       WHERE c.customer_id = p.customer_id
       ) first_name,
       (SELECT c.last_name FROM sakila.customer c #2
        WHERE c.customer_id = p.customer_id
        ) last_name,
        (SELECT ct.city							   #3
         FROM sakila.customer c 
         INNER JOIN sakila.address a
            ON c.address_id = a.address_id
		 INNER JOIN sakila.city ct
			ON a.city_id = ct.city_id
		 WHERE c.customer_id = p.customer_id #we used 3 subqueries because scalar subqueries can only return a single column and row.
         ) city,
         sum(p.amount) tot_payments,
		 count(*) tot_rentals
	   FROM sakila.payment p
       GROUP BY p.customer_id
       #There are two main differences between this query and the earlier version using a subquery
       #in the from clause
       #	1-instead of joining the customer, address, and city tables to the payment data, correlated scalar
       #		subqueries are used in the select clause to look up the customer's first/last names and city.
       #	2-The customer table is accessed three times (once in each od the three subqueries) rather than just one.
       
       #Scalar Subqueries can also appear in the ORDER BY clause
       SELECT a.actor_id , a.first_name, a.last_name
       FROM sakila.actor a
       ORDER BY 
         (SELECT count(*) FROM sakila.film_actor fa #The subquery counts the number of datapoints with th
          WHERE fa.actor_id = a.actor_id) DESC       #and returns the count per actor id, then ORDER BY
													 #
       
