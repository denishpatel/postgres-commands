# postgres-commands
List of postgres commands

# Find INVALID indices

SELECT schemaname || '.' || relname AS table, indexrelname AS index, pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size, indisvalid AS is_index_valid
FROM pg_stat_user_indexes ui 
   JOIN pg_index i ON ui.indexrelid =i.indexrelid WHERE 
    indisvalid is false; 
    
