SQLSTATE[23000]: Integrity constraint violation: 1062 Duplicate entry '1-? test' for key 'user_away_reasons.user_away_reasons_account_id_name_unique' (SQL: insert into `user_away_reasons` (`account_id`, `name`, `extra_attributes`, `sort_order`, `id`, `updated_at`, `created_at`) values (1, ![:smile:](https://a.slack-edge.com/production-standard-emoji-assets/16.0/apple-medium/1f604@2x.png) test, ?, 7, 985980293435494401, 2026-06-13 18:53:40, 2026-06-13 18:53:40))

667 |         // If an exception occurs when attempting to run a query, we'll format the error  
668 |         // message to include the bindings with SQL, which will make this exception a  
669 |         // lot more helpful to the developer instead of just the database's errors.  
670 |         catch (Exception $e) {  

671 |             throw new QueryException(

672 |                 $query, $this->prepareBindings($bindings), $e  
673 |             );  
674 |         }  
675 |   
676 |         return $result;


![[2026-06-13-001070.png]]