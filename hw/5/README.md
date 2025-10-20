


db.createUser({user:"admin", pwd:"secret", roles:[{role: "userAdminAnyDatabase", db:"admin"}, {role:"clusterAdmin", db:"admin"}, {role:"readWriteAnyDatabase", db:"admin"}, {role:"dbAdminAnyDatabase", db:"admin"}]})




root@dev-mongo-0:~# mongosh -u admin -p secret --authenticationDatabase=admin1
Current Mongosh Log ID:	68f4ef6710ab96d98880d710





root@dev-mongo-0:~# mongoimport --port 27017 -d premier-league -c season11-12 --file /home/pirogov/csvjson.json --jsonArray --host 10.0.20.1
2025-10-19T10:15:36.601-0400	connected to: mongodb://10.0.20.1:27017/
2025-10-19T10:15:36.629-0400	Failed: (Unauthorized) Command insert requires authentication

now with authmong

root@dev-mongo-0:~# mongoimport --port 27017 -d premier-league -c season11-12 --file /home/pirogov/csvjson.json --jsonArray --host 10.0.20.1 -u admin -p secret --authenticationDatabase=admin
2025-10-19T10:16:08.732-0400	connected to: mongodb://10.0.20.1:27017/
2025-10-19T10:16:08.973-0400	380 document(s) imported successfully. 0 document(s) failed to import.


root@dev-mongo-0:~# mongoexport --collection=season11_12 --db=premier-league --out=events.json
2025-10-19T10:19:09.272-0400	connected to: mongodb://localhost/
2025-10-19T10:19:09.273-0400	Failed: (Unauthorized) Command listCollections requires authentication

root@dev-mongo-0:~# mongoexport --collection=season11_12 --db=premier-league --out=pl_season_11_12.json -u admin -p secret --authenticationDatabase=admin
2025-10-19T10:19:46.711-0400	connected to: mongodb://localhost/
2025-10-19T10:19:46.729-0400	exported 380 records


mongo-perf

1 cpu 4gb ram



4 cpu 8gb ram
