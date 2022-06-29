# SQL Query examples (MySQL workbench + Command client, SQL Server, Oracle)

Example Database SAKILA found at https://dev.mysql.com/


Analysis using data windows, basic clause, sub-queries, joins, and others.

PERCENT TOTAL OF OVERALL SALES BY COUNTRY 

          SELECT B.country, B.tot_per_country, B.percent_total
          FROM 
          (
          SELECT co.country, count(co.country) tot_per_country, 
	  sum(count(co.country)) over () tot_overall_countries, 
          concat(((count(co.country) / sum(count(co.country)) over ()) * 100), '%') percent_total
          FROM sakila.address a
            INNER JOIN sakila.city ct ON a.city_id = ct.city_id
            INNER JOIN sakila.country co ON ct.country_id = co.country_id
          GROUP BY co.country
          ) B
          HAVING B.tot_per_country > 1 #must have more than 1 sale per country
          ORDER BY B.tot_per_country desc
          
	  
 ![percent_total_sales_by_country](https://user-images.githubusercontent.com/67971912/176245207-b61a6d48-fb7b-4cf3-8c2d-8b1908ed9c77.png)


WHAT CATEGORIES DO MOST ACTORS/ACTRESSES REPRESENT? SHOW THE BIGGEST ONE
SHOWS ACTOR/ACTRESS full name ALONG WITH THE CATEGORY OF FILM WHICH HAS THE MAX PERCENTAGE OF TOTAL MOVIES FOR ALL CATEGORIES.

		SELECT B.full_name, B.name category, B.num_films, B.tot_films,
			CONCAT(max(B.percent_starring_roles), '%') highest_starring_film_percentage 
		FROM 
		(
		SELECT CONCAT(a.first_name,' ', a.last_name) full_name , c.name, 
			count(c.name) num_films, sum(count(c.name)) over (partition by concat(a.first_name, ' ',a.last_name)) tot_films,
			ROUND(((count(c.name) / sum(count(c.name)) over 
				(partition by concat(a.first_name, ' ',a.last_name)) ) * 100), 1) percent_starring_roles
		FROM sakila.actor a
		INNER JOIN sakila.film_actor fa ON a.actor_id = fa.actor_id
		INNER JOIN sakila.film f ON fa.film_id = f.film_id
		INNER JOIN sakila.film_category fc ON f.film_id = fc.film_id
		INNER JOIN sakila.category c ON fc.category_id = c.category_id
		GROUP BY a.first_name, a.last_name , c.name
		ORDER BY full_name desc, num_films desc
		) B
		GROUP BY B.full_name
		ORDER BY B.full_name asc
        
	
![max_percentage_actr_by_genre](https://user-images.githubusercontent.com/67971912/176245583-9b03e885-932d-40db-840a-285d37cbed32.png)

WHICH COUNTRIES HAVE THE HIGHEST RANKING BY NUMBER OF RENTALS AND FILM TITLE? (COMMON TABLE EXPRESSION, SUB-		QUERIES, RANKINGS, DATA WINDOWS)

SELECT B.title, B.country, B.max_by_country, B.tot_num_by_title
                        FROM 
                        (
                        SELECT A.title , A.country, A.max_by_country, A.tot_num_by_title,
                                #dense_rank() over (partition by A.title order by A.max_by_country desc) dense_ranking,
                                #rank() over (partition by A.title order by A.max_by_country desc) ranking,
                                row_number() over (partition by A.title order by A.max_by_country desc) row_number_rank
                        FROM (
                        
                        WITH cust_adrs_rent_film AS (
                        SELECT customer_id, inventory_id, rental_id , c.first_name, c.last_name,
								c.address_id, cy.city, ct.country, f.title, f.film_id
                        FROM sakila.rental
                        INNER JOIN sakila.customer c USING (customer_id)
                        INNER JOIN sakila.address a USING (address_id)
                        INNER JOIN sakila.city cy USING (city_id)
                        INNER JOIN sakila.country ct USING (country_id)
                        INNER JOIN sakila.inventory i USING (inventory_id)
                        INNER JOIN sakila.film f USING (film_id)
                        ), 
                        cust_inv_rental_id AS
                        (SELECT carf.title, carf.country, count(*) over (partition by  carf.film_id ,
						carf.country) max_by_country ,
						count(*) over (partition by carf.title, carf.film_id) tot_num_by_title
                         FROM cust_adrs_rent_film carf 
                         ) 
                         SELECT ciri.title, ciri.country,  MAX(ciri.max_by_country) max_by_country, tot_num_by_title
								
                         FROM cust_inv_rental_id ciri
                         GROUP BY ciri.title, ciri.country
                         ) A
                         ORDER BY  A.title
                         )B
                         WHERE B.row_number_rank = 1

		    
		    
![film_ranking_by_country](https://user-images.githubusercontent.com/67971912/176557989-2d3f7d19-b8ca-4b08-919b-b09bc186345a.png)
