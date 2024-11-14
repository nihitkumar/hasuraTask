# Task Two :

## Describe any issues you discovered with the shared deployment artifacts and thesteps you took to remediate the issue(s).

Various updates and adjustments were made to the provided deployment configuration to improve functionality and address access issues.

1. **Image Version for graphql-engine:**
The original image version ```(hasura/graphql-engine:v2.48.2-pro)``` appeared to be a pro version hosted in a private registry, for which we lack access credentials. This was changed to the open-source version ```hasura/graphql-engine:v2.42.0```.

2. **Port Mapping for graphql-engine:**
The initial setup mapped port ```80``` to ```8111``` on the host, but the Hasura console actually runs on port 8080. The configuration was updated to map port ```8080``` to ```8111```.

3. **Database URL Correction:**
The HASURA_GRAPHQL_METADATA_DATABASE_URL contained an incorrect password ```(somepassword)``` in the URL ```(postgres://postgres:somepassword@postgresdb:5432/postgres)```. This was corrected to ```postgres://postgres:postgrespassword@postgres:5432/postgres```, aligning with the saved ```POSTGRES_PASSWORD: postgrespassword```.

4. **Additional Volume for Metadata:**
A new volume ```hasura-metadata``` was added, allowing Hasura to store and import metadata as needed for the configuration.

5. **Custom Network Addition:**
A custom network, ```my-network```, was added with the ```bridge``` driver to facilitate isolated communication between all services in the deployment.

## Include the requested artifacts from each above question

Q.1

GraphQl :
```
# Execute as an administrator

query getTracks($genre: String, $limit: Int, $offset: Int) {
  track(limit: $limit, offset: $offset, where: {genre: {name: {_eq: $genre}}}) {
    name
    track_id
  }
}

```

Query Variable :
```
{
"genre":"Metal",
"limit": 5,
"offset":50
}
```
Output :

```
{
  "data": {
    "track": [
      {
        "name": "Trupets Of Jericho",
        "track_id": 190
      },
      {
        "name": "Machine Men",
        "track_id": 191
      },
      {
        "name": "The Alchemist",
        "track_id": 192
      },
      {
        "name": "Realword",
        "track_id": 193
      },
      {
        "name": "Free Speech For The Dumb",
        "track_id": 408
      }
    ]
  }
}
```
![](./images/Q1.png)


Q.2

GraphQl :
```
# Execute as an Artist

query getAlbumsAsArtist {
  album {
    title
  }
}

```

Output :

```
{
  "data": {
    "album": [
      {
        "title": "Balls to the Wall"
      },
      {
        "title": "Restless and Wild"
      }
    ]
  }
}
```

![](./images/%20Q2.png)

Q3.

GraphQl :
```
# Execute as an Artist

query trackValue {
  track_aggregate {
    aggregate {
      sum {
        unit_price
      }
    }
  }
}

```
Output :
```
{
  "data": {
    "track_aggregate": {
      "aggregate": {
        "sum": {
          "unit_price": 3680.97
        }
      }
    }
  }
}
```
![](./images/Q3.png)
## Execute a complex query of your choice, with and without caching. Share the query, there sponse and the response time for each.


GraphQl :

```
query getTracksWithAlbumsAndGenres($genre: String, $limit: Int, $offset: Int) {
  track(limit: $limit, offset: $offset, where: {genre: {name: {_eq: $genre}}}) {
    track_id
    name
    album {
      title
    }
    genre {
      name
    }
  }
}
```
Query Variable :

```
{
  "genre": "Rock",
  "limit": 10,
  "offset": 0
}
```

Output :

```
{
  "genre": "Rock",
  "limit": 10,
  "offset": 0
}
```


| Query |	Caching	Response | Time (ms) |
|-------|--------------------|-----------|
| getTracksWithAlbumsAndGenres | Yes| 52 |
| getTracksWithAlbumsAndGenres |No|69|

**With cache enabled :**

![](./images/cache.png)

**Without cache enabled:**

![](./images/no-cache.png)

## Describe any challenges you encountered when executing the above queries andyour troubleshooting steps to address them.

When executing the queries, I faced a few challenges:

1. **Permission Issues :** As the "Artist" role, I lacked access to certain tables, leading to authorization errors. I resolved this by adjusting the role permissions in the Hasura Console to allow read access for specific tables.

2. **Variable Mismatch :** For the getTracks query, variable names or types sometimes caused errors. I double-checked and matched variable names and types, using test values to ensure they were correct.

3. **Metadata Reload:** Changes to metadata or permissions didn't apply immediately. I fixed this by reloading metadata from the Console under "Settings" or via CLI.

4. **Issue with Importing the Chinook Database :**

Here’s a step-by-step explanation of how to import the Chinook database into your Postgres container and connect it to Hasura:

- Copy the Chinook SQL file into the Postgres container:
```
docker cp Chinook_PostgreSql.sql <postgres_container_name>:/postgres.sql
```
This command copies the Chinook_PostgreSql.sql file from your local machine into the task-two-artifacts-postgres-1 container at the /postgres.sql path.

- Access the Postgres CLI within the container:
```
docker exec -it <postgres_container_name> psql -U postgres
```
This command starts a psql session within the Postgres container as the postgres user.

- Run the SQL file to create and populate the Chinook database:
```
\i /postgres.sql
```
This command instructs psql to execute the SQL script, which drops any existing Chinook database, creates a new one, and populates it with tables and data. After execution, tables like album, artist, and track are created and populated.

- Verify the tables in the Chinook database:
```
\dt
```
This command lists the tables within the chinook database to confirm that the import was successful.

- Connect Hasura to the Chinook database:

In the Hasura Console, go to Data > Manage > Connect Database.
Enter a database name and select "Connect via Database URL."
Use the database URL as specified in your Docker Compose configuration:

```
postgres://postgres:postgrespassword@postgres:5432/chinook
```
