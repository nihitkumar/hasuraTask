## Briefly describe the steps taken to configure your Hasura GraphQL Engine

1. Get the Compose file from our repo. If you're using curl, run this command in a new directory:

```
curl https://raw.githubusercontent.com/hasura/graphql-engine/stable/install-manifests/docker-compose/docker-compose.yaml -o docker-compose.yml
```

2. Used the edited docker-compose file configures a PostgreSQL database and Hasura GraphQL Engine, connecting Hasura to PostgreSQL to offer a GraphQL interface. It includes an SQL initialization file, sets up persistent storage, and enables detailed logging and development mode for easier debugging.

```
services:
  postgres:
    image: postgres:15
    restart: always
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./chinook-postgresql.sql/Chinook_PostgreSql.sql:/docker-entrypoint-initdb.d/Chinook_PostgreSql.sql
    environment:
      POSTGRES_PASSWORD: postgrespassword
  graphql-engine:
    image: hasura/graphql-engine:v2.42.0
    ports:
      - "8080:8080"
    restart: always
    environment:
      HASURA_GRAPHQL_METADATA_DATABASE_URL: postgres://postgres:postgrespassword@postgres:5432/postgres
      PG_DATABASE_URL: postgres://postgres:postgrespassword@postgres:5432/postgres
      HASURA_GRAPHQL_ENABLE_CONSOLE: "true"
      HASURA_GRAPHQL_DEV_MODE: "true"
      HASURA_GRAPHQL_ENABLED_LOG_TYPES: startup, http-log, webhook-log, websocket-log, query-log
    depends_on:
      - postgres
volumes:
  db_data:

```
*Note that to import the Chinook database, we use the following command:*

```
./chinook-postgresql.sql/Chinook_PostgreSql.sql:/docker-entrypoint-initdb.d/Chinook_PostgreSql.sql
```
*This command ensures that, upon startup, the PostgreSQL container automatically loads and executes the Chinook_PostgreSql.sql script, initializing the Chinook database.*


3. Then, run the following command to start both the Hasura GraphQL Engine and the Postgres database in Docker containers:

```
docker compose up -d
```

4. Log in to the console

```
http://localhost:8080
```

5. Then, navigate to: Data > Manage > Connect > postgres. Enter a database name and select the option to connect via Database URL.Use the following database URL, as configured in the docker-compose.yml file:

```
postgres://postgres:postgrespassword@postgres:5432/chinook
```

## Assume the client uses the x-hasura-artist-id header for artist identification, ensuring artists can only access albums assigned to them.

- Go to the Data Tab: In the Hasura Console, go to the Data section and locate the albums table.

- Set Up a Role for Artists: Click on Permissions for the albums table. If you don't already have a role for artists, create a role called artist.

- Define Select Permissions: Select the artist role and configure Select permissions for the role.In the Row Select Permissions section, set the rule to restrict access by matching the album's artist_id with the x-hasura-artist-id header:

```
{
  "artist_id": {
    "_eq": "X-Hasura-Artist-Id"
  }
}
```
This rule ensures that the artist can only view albums where the artist_id matches the x-hasura-artist-id header passed in the request

![](./images/role.png)

## Describe any challenges you encountered when executing the above queries and your troubleshooting steps to address them.

1. **Setting Up Hasura with Docker Compose:** Configuring the Docker Compose setup for Hasura and Postgres was tricky. I fixed it by reviewing the official documentation and adjusting environment variables for correct database connections. After verifying with docker-compose logs, I successfully set up the services.

2. **Configuring Permissions with x-hasura-artist-id:** Setting row-level security based on the x-hasura-artist-id header was complex. I tested basic permissions first and then set up dynamic permissions using the header, ensuring only relevant albums were accessible based on the artist's ID.

3. **Writing GraphQl Queries for Analytics:** Writing queries to return the artist with the most albums, top genres, and track counts required understanding table relationships and using aggregation functions. I tested and optimized each query step-by-step for accurate results.




## Using a Postgres client, configure a SQL statement to retrieve the below information.

1. Return the artist with the most number of albums
```
chinook=# SELECT artist.artist_id, artist.name, COUNT(album.album_id) AS album_count
FROM album
JOIN artist ON album.artist_id = artist.artist_id
GROUP BY artist.artist_id
ORDER BY album_count DESC
LIMIT 1;
 artist_id |    name     | album_count 
-----------+-------------+-------------
        90 | Iron Maiden |          21
```

2. Return the top three genres found in the dataset in descending order
```
chinook=# SELECT genre.name, COUNT(album.album_id) AS album_count
FROM album
JOIN track ON album.album_id = track.album_id
JOIN genre ON track.genre_id = genre.genre_id
GROUP BY genre.genre_id
ORDER BY album_count DESC
LIMIT 3;
 name  | album_count 
-------+-------------
 Rock  |        1297
 Latin |         579
 Metal |         374
(3 rows)

```

3. Return the number of tracks and average run time for each media type
```
chinook=# SELECT media_type.name, 
       COUNT(track.track_id) AS track_count,
       AVG(track.milliseconds) AS avg_runtime
FROM track
JOIN media_type ON track.media_type_id = media_type.media_type_id
GROUP BY media_type.media_type_id;
            name             | track_count |     avg_runtime      
-----------------------------+-------------+----------------------
 AAC audio file              |          11 |  276506.909090909091
 Purchased AAC audio file    |           7 |  260894.714285714286
 Protected AAC audio file    |         237 |  281723.873417721519
 MPEG audio file             |        3034 |  265574.288727752142
 Protected MPEG-4 video file |         214 | 2342940.425233644860
(5 rows)

```

## GraphQL query and results set for each of the below statements

1. How many artists are in the database?

GraphQl :
```
query {
  artist_aggregate {
    aggregate {
      count
    }
  }
}
```
Output :
```
{
  "data": {
    "artist_aggregate": {
      "aggregate": {
        "count": 276
      }
    }
  }
}
```

2. List the first track of every album by every artist in ascending order.

QraphQl :
```
query{
  artist (order_by :{artist_id : asc}){
    artist_id
    name,
    albums{
      title
		tracks(order_by: {track_id :asc} limit: 1){
      name
    }
      
  }
}
}
```
Output : 
```
{
  "data": {
    "artist": [
      {
        "artist_id": 1,
        "name": "AC/DC",
        "albums": [
          {
            "title": "For Those About To Rock We Salute You",
            "tracks": [
              {
                "name": "For Those About To Rock (We Salute You)"
              }
            ]
          },
          {
            "title": "Let There Be Rock",
            "tracks": [
              {
                "name": "Go Down"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 2,
        "name": "Accept",
        "albums": [
          {
            "title": "Balls to the Wall",
            "tracks": [
              {
                "name": "Balls to the Wall"
              }
            ]
          },
          {
            "title": "Restless and Wild",
            "tracks": [
              {
                "name": "Fast As a Shark"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 3,
        "name": "Aerosmith",
        "albums": [
          {
            "title": "Big Ones",
            "tracks": [
              {
                "name": "Walk On Water"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 4,
        "name": "Alanis Morissette",
        "albums": [
          {
            "title": "Jagged Little Pill",
            "tracks": [
              {
                "name": "All I Really Want"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 5,
        "name": "Alice In Chains",
        "albums": [
          {
            "title": "Facelift",
            "tracks": [
              {
                "name": "We Die Young"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 6,
        "name": "Antônio Carlos Jobim",
        "albums": [
          {
            "title": "Warner 25 Anos",
            "tracks": [
              {
                "name": "Desafinado"
              }
            ]
          },
          {
            "title": "Chill: Brazil (Disc 2)",
            "tracks": [
              {
                "name": "Garota De Ipanema"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 7,
        "name": "Apocalyptica",
        "albums": [
          {
            "title": "Plays Metallica By Four Cellos",
            "tracks": [
              {
                "name": "Enter Sandman"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 8,
        "name": "Audioslave",
        "albums": [
          {
            "title": "Audioslave",
            "tracks": [
              {
                "name": "Cochise"
              }
            ]
          },
          {
            "title": "Out Of Exile",
            "tracks": [
              {
                "name": "Your Time Has Come"
              }
            ]
          },
          {
            "title": "Revelations",
            "tracks": [
              {
                "name": "Revelations"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 9,
        "name": "BackBeat",
        "albums": [
          {
            "title": "BackBeat Soundtrack",
            "tracks": [
              {
                "name": "Money"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 10,
        "name": "Billy Cobham",
        "albums": [
          {
            "title": "The Best Of Billy Cobham",
            "tracks": [
              {
                "name": "Quadrant"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 11,
        "name": "Black Label Society",
        "albums": [
          {
            "title": "Alcohol Fueled Brewtality Live! [Disc 1]",
            "tracks": [
              {
                "name": "Intro/ Low Down"
              }
            ]
          },
          {
            "title": "Alcohol Fueled Brewtality Live! [Disc 2]",
            "tracks": [
              {
                "name": "Heart Of Gold"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 12,
        "name": "Black Sabbath",
        "albums": [
          {
            "title": "Black Sabbath",
            "tracks": [
              {
                "name": "Black Sabbath"
              }
            ]
          },
          {
            "title": "Black Sabbath Vol. 4 (Remaster)",
            "tracks": [
              {
                "name": "Wheels Of Confusion / The Straightener"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 13,
        "name": "Body Count",
        "albums": [
          {
            "title": "Body Count",
            "tracks": [
              {
                "name": "Smoked Pork"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 14,
        "name": "Bruce Dickinson",
        "albums": [
          {
            "title": "Chemical Wedding",
            "tracks": [
              {
                "name": "King In Crimson"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 15,
        "name": "Buddy Guy",
        "albums": [
          {
            "title": "The Best Of Buddy Guy - The Millenium Collection",
            "tracks": [
              {
                "name": "First Time I Met The Blues"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 16,
        "name": "Caetano Veloso",
        "albums": [
          {
            "title": "Prenda Minha",
            "tracks": [
              {
                "name": "Jorge Da Capadócia"
              }
            ]
          },
          {
            "title": "Sozinho Remix Ao Vivo",
            "tracks": [
              {
                "name": "Sozinho (Hitmakers Classic Mix)"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 17,
        "name": "Chico Buarque",
        "albums": [
          {
            "title": "Minha Historia",
            "tracks": [
              {
                "name": "Carolina"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 18,
        "name": "Chico Science & Nação Zumbi",
        "albums": [
          {
            "title": "Afrociberdelia",
            "tracks": [
              {
                "name": "Mateus Enter"
              }
            ]
          },
          {
            "title": "Da Lama Ao Caos",
            "tracks": [
              {
                "name": "Banditismo Por Uma Questa"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 19,
        "name": "Cidade Negra",
        "albums": [
          {
            "title": "Acústico MTV [Live]",
            "tracks": [
              {
                "name": "Girassol"
              }
            ]
          },
          {
            "title": "Cidade Negra - Hits",
            "tracks": [
              {
                "name": "Onde Você Mora?"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 20,
        "name": "Cláudio Zoli",
        "albums": [
          {
            "title": "Na Pista",
            "tracks": [
              {
                "name": "Noite Do Prazer"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 21,
        "name": "Various Artists",
        "albums": [
          {
            "title": "Axé Bahia 2001",
            "tracks": [
              {
                "name": "Dig-Dig, Lambe-Lambe (Ao Vivo)"
              }
            ]
          },
          {
            "title": "Carnaval 2001",
            "tracks": [
              {
                "name": "Vai-Vai 2001"
              }
            ]
          },
          {
            "title": "Sambas De Enredo 2001",
            "tracks": [
              {
                "name": "Imperatriz"
              }
            ]
          },
          {
            "title": "Vozes do MPB",
            "tracks": [
              {
                "name": "Linha de Passe (João Bosco)"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 22,
        "name": "Led Zeppelin",
        "albums": [
          {
            "title": "BBC Sessions [Disc 1] [Live]",
            "tracks": [
              {
                "name": "You Shook Me"
              }
            ]
          },
          {
            "title": "Physical Graffiti [Disc 1]",
            "tracks": [
              {
                "name": "Custard Pie"
              }
            ]
          },
          {
            "title": "BBC Sessions [Disc 2] [Live]",
            "tracks": [
              {
                "name": "Immigrant Song"
              }
            ]
          },
          {
            "title": "Coda",
            "tracks": [
              {
                "name": "We're Gonna Groove"
              }
            ]
          },
          {
            "title": "Houses Of The Holy",
            "tracks": [
              {
                "name": "The Song Remains The Same"
              }
            ]
          },
          {
            "title": "In Through The Out Door",
            "tracks": [
              {
                "name": "In The Evening"
              }
            ]
          },
          {
            "title": "IV",
            "tracks": [
              {
                "name": "Black Dog"
              }
            ]
          },
          {
            "title": "Led Zeppelin I",
            "tracks": [
              {
                "name": "Good Times Bad Times"
              }
            ]
          },
          {
            "title": "Led Zeppelin II",
            "tracks": [
              {
                "name": "Whole Lotta Love"
              }
            ]
          },
          {
            "title": "Led Zeppelin III",
            "tracks": [
              {
                "name": "Immigrant Song"
              }
            ]
          },
          {
            "title": "Physical Graffiti [Disc 2]",
            "tracks": [
              {
                "name": "In The Light"
              }
            ]
          },
          {
            "title": "Presence",
            "tracks": [
              {
                "name": "Achilles Last Stand"
              }
            ]
          },
          {
            "title": "The Song Remains The Same (Disc 1)",
            "tracks": [
              {
                "name": "Rock & Roll"
              }
            ]
          },
          {
            "title": "The Song Remains The Same (Disc 2)",
            "tracks": [
              {
                "name": "No Quarter"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 23,
        "name": "Frank Zappa & Captain Beefheart",
        "albums": [
          {
            "title": "Bongo Fury",
            "tracks": [
              {
                "name": "Debra Kadabra"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 24,
        "name": "Marcos Valle",
        "albums": [
          {
            "title": "Chill: Brazil (Disc 1)",
            "tracks": [
              {
                "name": "Guanabara"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 25,
        "name": "Milton Nascimento & Bebeto",
        "albums": []
      },
      {
        "artist_id": 26,
        "name": "Azymuth",
        "albums": []
      },
      {
        "artist_id": 27,
        "name": "Gilberto Gil",
        "albums": [
          {
            "title": "As Canções de Eu Tu Eles",
            "tracks": [
              {
                "name": "Óia Eu Aqui De Novo"
              }
            ]
          },
          {
            "title": "Quanta Gente Veio Ver (Live)",
            "tracks": [
              {
                "name": "Introdução (Live)"
              }
            ]
          },
          {
            "title": "Quanta Gente Veio ver--Bônus De Carnaval",
            "tracks": [
              {
                "name": "Doce De Carnaval (Candy All)"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 28,
        "name": "João Gilberto",
        "albums": []
      },
      {
        "artist_id": 29,
        "name": "Bebel Gilberto",
        "albums": []
      },
      {
        "artist_id": 30,
        "name": "Jorge Vercilo",
        "albums": []
      },
      {
        "artist_id": 31,
        "name": "Baby Consuelo",
        "albums": []
      },
      {
        "artist_id": 32,
        "name": "Ney Matogrosso",
        "albums": []
      },
      {
        "artist_id": 33,
        "name": "Luiz Melodia",
        "albums": []
      },
      {
        "artist_id": 34,
        "name": "Nando Reis",
        "albums": []
      },
      {
        "artist_id": 35,
        "name": "Pedro Luís & A Parede",
        "albums": []
      },
      {
        "artist_id": 36,
        "name": "O Rappa",
        "albums": [
          {
            "title": "Radio Brasil (O Som da Jovem Vanguarda) - Seleccao de Henrique Amaro",
            "tracks": [
              {
                "name": "Instinto Colectivo"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 37,
        "name": "Ed Motta",
        "albums": [
          {
            "title": "The Best of Ed Motta",
            "tracks": [
              {
                "name": "Solução"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 38,
        "name": "Banda Black Rio",
        "albums": []
      },
      {
        "artist_id": 39,
        "name": "Fernanda Porto",
        "albums": []
      },
      {
        "artist_id": 40,
        "name": "Os Cariocas",
        "albums": []
      },
      {
        "artist_id": 41,
        "name": "Elis Regina",
        "albums": [
          {
            "title": "Elis Regina-Minha História",
            "tracks": [
              {
                "name": "O Bêbado e a Equilibrista"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 42,
        "name": "Milton Nascimento",
        "albums": [
          {
            "title": "Milton Nascimento Ao Vivo",
            "tracks": [
              {
                "name": "Coração De Estudante"
              }
            ]
          },
          {
            "title": "Minas",
            "tracks": [
              {
                "name": "Minas"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 43,
        "name": "A Cor Do Som",
        "albums": []
      },
      {
        "artist_id": 44,
        "name": "Kid Abelha",
        "albums": []
      },
      {
        "artist_id": 45,
        "name": "Sandra De Sá",
        "albums": []
      },
      {
        "artist_id": 46,
        "name": "Jorge Ben",
        "albums": [
          {
            "title": "Jorge Ben Jor 25 Anos",
            "tracks": [
              {
                "name": "Engenho De Dentro"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 47,
        "name": "Hermeto Pascoal",
        "albums": []
      },
      {
        "artist_id": 48,
        "name": "Barão Vermelho",
        "albums": []
      },
      {
        "artist_id": 49,
        "name": "Edson, DJ Marky & DJ Patife Featuring Fernanda Porto",
        "albums": []
      },
      {
        "artist_id": 50,
        "name": "Metallica",
        "albums": [
          {
            "title": "Garage Inc. (Disc 1)",
            "tracks": [
              {
                "name": "Free Speech For The Dumb"
              }
            ]
          },
          {
            "title": "Black Album",
            "tracks": [
              {
                "name": "Enter Sandman"
              }
            ]
          },
          {
            "title": "Garage Inc. (Disc 2)",
            "tracks": [
              {
                "name": "Helpless"
              }
            ]
          },
          {
            "title": "Kill 'Em All",
            "tracks": [
              {
                "name": "Hit The Lights"
              }
            ]
          },
          {
            "title": "Load",
            "tracks": [
              {
                "name": "Ain't My Bitch"
              }
            ]
          },
          {
            "title": "Master Of Puppets",
            "tracks": [
              {
                "name": "Battery"
              }
            ]
          },
          {
            "title": "ReLoad",
            "tracks": [
              {
                "name": "Fuel"
              }
            ]
          },
          {
            "title": "Ride The Lightning",
            "tracks": [
              {
                "name": "Fight Fire With Fire"
              }
            ]
          },
          {
            "title": "St. Anger",
            "tracks": [
              {
                "name": "Frantic"
              }
            ]
          },
          {
            "title": "...And Justice For All",
            "tracks": [
              {
                "name": "Blackened"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 51,
        "name": "Queen",
        "albums": [
          {
            "title": "Greatest Hits II",
            "tracks": [
              {
                "name": "A Kind Of Magic"
              }
            ]
          },
          {
            "title": "Greatest Hits I",
            "tracks": [
              {
                "name": "Bohemian Rhapsody"
              }
            ]
          },
          {
            "title": "News Of The World",
            "tracks": [
              {
                "name": "We Will Rock You"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 52,
        "name": "Kiss",
        "albums": [
          {
            "title": "Greatest Kiss",
            "tracks": [
              {
                "name": "Detroit Rock City"
              }
            ]
          },
          {
            "title": "Unplugged [Live]",
            "tracks": [
              {
                "name": "Comin' Home"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 53,
        "name": "Spyro Gyra",
        "albums": [
          {
            "title": "Heart of the Night",
            "tracks": [
              {
                "name": "Heart of the Night"
              }
            ]
          },
          {
            "title": "Morning Dance",
            "tracks": [
              {
                "name": "Morning Dance"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 54,
        "name": "Green Day",
        "albums": [
          {
            "title": "International Superhits",
            "tracks": [
              {
                "name": "Maria"
              }
            ]
          },
          {
            "title": "American Idiot",
            "tracks": [
              {
                "name": "American Idiot"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 55,
        "name": "David Coverdale",
        "albums": [
          {
            "title": "Into The Light",
            "tracks": [
              {
                "name": "Into The Light"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 56,
        "name": "Gonzaguinha",
        "albums": [
          {
            "title": "Meus Momentos",
            "tracks": [
              {
                "name": "Grito De Alerta"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 57,
        "name": "Os Mutantes",
        "albums": [
          {
            "title": "Minha História",
            "tracks": [
              {
                "name": "Balada Do Louco"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 58,
        "name": "Deep Purple",
        "albums": [
          {
            "title": "MK III The Final Concerts [Disc 1]",
            "tracks": [
              {
                "name": "Burn"
              }
            ]
          },
          {
            "title": "The Final Concerts (Disc 2)",
            "tracks": [
              {
                "name": "Space Truckin'"
              }
            ]
          },
          {
            "title": "Come Taste The Band",
            "tracks": [
              {
                "name": "Comin' Home"
              }
            ]
          },
          {
            "title": "Deep Purple In Rock",
            "tracks": [
              {
                "name": "Speed King"
              }
            ]
          },
          {
            "title": "Fireball",
            "tracks": [
              {
                "name": "Fireball"
              }
            ]
          },
          {
            "title": "Knocking at Your Back Door: The Best Of Deep Purple in the 80's",
            "tracks": [
              {
                "name": "Knocking At Your Back Door"
              }
            ]
          },
          {
            "title": "Machine Head",
            "tracks": [
              {
                "name": "Highway Star"
              }
            ]
          },
          {
            "title": "Purpendicular",
            "tracks": [
              {
                "name": "Vavoom : Ted The Mechanic"
              }
            ]
          },
          {
            "title": "Slaves And Masters",
            "tracks": [
              {
                "name": "King Of Dreams"
              }
            ]
          },
          {
            "title": "Stormbringer",
            "tracks": [
              {
                "name": "Stormbringer"
              }
            ]
          },
          {
            "title": "The Battle Rages On",
            "tracks": [
              {
                "name": "The Battle Rages On"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 59,
        "name": "Santana",
        "albums": [
          {
            "title": "Supernatural",
            "tracks": [
              {
                "name": "(Da Le) Yaleo"
              }
            ]
          },
          {
            "title": "Santana - As Years Go By",
            "tracks": [
              {
                "name": "Jingo"
              }
            ]
          },
          {
            "title": "Santana Live",
            "tracks": [
              {
                "name": "Evil Ways"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 60,
        "name": "Santana Feat. Dave Matthews",
        "albums": []
      },
      {
        "artist_id": 61,
        "name": "Santana Feat. Everlast",
        "albums": []
      },
      {
        "artist_id": 62,
        "name": "Santana Feat. Rob Thomas",
        "albums": []
      },
      {
        "artist_id": 63,
        "name": "Santana Feat. Lauryn Hill & Cee-Lo",
        "albums": []
      },
      {
        "artist_id": 64,
        "name": "Santana Feat. The Project G&B",
        "albums": []
      },
      {
        "artist_id": 65,
        "name": "Santana Feat. Maná",
        "albums": []
      },
      {
        "artist_id": 66,
        "name": "Santana Feat. Eagle-Eye Cherry",
        "albums": []
      },
      {
        "artist_id": 67,
        "name": "Santana Feat. Eric Clapton",
        "albums": []
      },
      {
        "artist_id": 68,
        "name": "Miles Davis",
        "albums": [
          {
            "title": "The Essential Miles Davis [Disc 1]",
            "tracks": [
              {
                "name": "Now's The Time"
              }
            ]
          },
          {
            "title": "The Essential Miles Davis [Disc 2]",
            "tracks": [
              {
                "name": "My Funny Valentine (Live)"
              }
            ]
          },
          {
            "title": "Miles Ahead",
            "tracks": [
              {
                "name": "Springsville"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 69,
        "name": "Gene Krupa",
        "albums": [
          {
            "title": "Up An' Atom",
            "tracks": [
              {
                "name": "Jeepers Creepers"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 70,
        "name": "Toquinho & Vinícius",
        "albums": [
          {
            "title": "Vinícius De Moraes - Sem Limite",
            "tracks": [
              {
                "name": "Samba Da Bênção"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 71,
        "name": "Vinícius De Moraes & Baden Powell",
        "albums": []
      },
      {
        "artist_id": 72,
        "name": "Vinícius De Moraes",
        "albums": [
          {
            "title": "Vinicius De Moraes",
            "tracks": [
              {
                "name": "Pela Luz Dos Olhos Teus"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 73,
        "name": "Vinícius E Qurteto Em Cy",
        "albums": []
      },
      {
        "artist_id": 74,
        "name": "Vinícius E Odette Lara",
        "albums": []
      },
      {
        "artist_id": 75,
        "name": "Vinicius, Toquinho & Quarteto Em Cy",
        "albums": []
      },
      {
        "artist_id": 76,
        "name": "Creedence Clearwater Revival",
        "albums": [
          {
            "title": "Chronicle, Vol. 1",
            "tracks": [
              {
                "name": "Susie Q"
              }
            ]
          },
          {
            "title": "Chronicle, Vol. 2",
            "tracks": [
              {
                "name": "Walking On The Water"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 77,
        "name": "Cássia Eller",
        "albums": [
          {
            "title": "Cássia Eller - Coleção Sem Limite [Disc 2]",
            "tracks": [
              {
                "name": "Gatas Extraordinárias"
              }
            ]
          },
          {
            "title": "Cássia Eller - Sem Limite [Disc 1]",
            "tracks": [
              {
                "name": "Malandragem"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 78,
        "name": "Def Leppard",
        "albums": [
          {
            "title": "Vault: Def Leppard's Greatest Hits",
            "tracks": [
              {
                "name": "Pour Some Sugar On Me"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 79,
        "name": "Dennis Chambers",
        "albums": [
          {
            "title": "Outbreak",
            "tracks": [
              {
                "name": "Roll Call"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 80,
        "name": "Djavan",
        "albums": [
          {
            "title": "Djavan Ao Vivo - Vol. 02",
            "tracks": [
              {
                "name": "Pétala"
              }
            ]
          },
          {
            "title": "Djavan Ao Vivo - Vol. 1",
            "tracks": [
              {
                "name": "Samurai"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 81,
        "name": "Eric Clapton",
        "albums": [
          {
            "title": "The Cream Of Clapton",
            "tracks": [
              {
                "name": "Layla"
              }
            ]
          },
          {
            "title": "Unplugged",
            "tracks": [
              {
                "name": "Signe"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 82,
        "name": "Faith No More",
        "albums": [
          {
            "title": "Album Of The Year",
            "tracks": [
              {
                "name": "Collision"
              }
            ]
          },
          {
            "title": "Angel Dust",
            "tracks": [
              {
                "name": "Land Of Sunshine"
              }
            ]
          },
          {
            "title": "King For A Day Fool For A Lifetime",
            "tracks": [
              {
                "name": "Get Out"
              }
            ]
          },
          {
            "title": "The Real Thing",
            "tracks": [
              {
                "name": "From Out Of Nowhere"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 83,
        "name": "Falamansa",
        "albums": [
          {
            "title": "Deixa Entrar",
            "tracks": [
              {
                "name": "Deixa Entrar"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 84,
        "name": "Foo Fighters",
        "albums": [
          {
            "title": "In Your Honor [Disc 1]",
            "tracks": [
              {
                "name": "In Your Honor"
              }
            ]
          },
          {
            "title": "In Your Honor [Disc 2]",
            "tracks": [
              {
                "name": "Still"
              }
            ]
          },
          {
            "title": "One By One",
            "tracks": [
              {
                "name": "All My Life"
              }
            ]
          },
          {
            "title": "The Colour And The Shape",
            "tracks": [
              {
                "name": "Doll"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 85,
        "name": "Frank Sinatra",
        "albums": [
          {
            "title": "My Way: The Best Of Frank Sinatra [Disc 1]",
            "tracks": [
              {
                "name": "My Way"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 86,
        "name": "Funk Como Le Gusta",
        "albums": [
          {
            "title": "Roda De Funk",
            "tracks": [
              {
                "name": "Entrando Na Sua (Intro)"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 87,
        "name": "Godsmack",
        "albums": [
          {
            "title": "Faceless",
            "tracks": [
              {
                "name": "Straight Out Of Line"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 88,
        "name": "Guns N' Roses",
        "albums": [
          {
            "title": "Appetite for Destruction",
            "tracks": [
              {
                "name": "Welcome to the Jungle"
              }
            ]
          },
          {
            "title": "Use Your Illusion I",
            "tracks": [
              {
                "name": "Right Next Door to Hell"
              }
            ]
          },
          {
            "title": "Use Your Illusion II",
            "tracks": [
              {
                "name": "Civil War"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 89,
        "name": "Incognito",
        "albums": [
          {
            "title": "Blue Moods",
            "tracks": [
              {
                "name": "Colibri"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 90,
        "name": "Iron Maiden",
        "albums": [
          {
            "title": "A Matter of Life and Death",
            "tracks": [
              {
                "name": "Different World"
              }
            ]
          },
          {
            "title": "A Real Dead One",
            "tracks": [
              {
                "name": "The Number Of The Beast"
              }
            ]
          },
          {
            "title": "A Real Live One",
            "tracks": [
              {
                "name": "Be Quick Or Be Dead"
              }
            ]
          },
          {
            "title": "Brave New World",
            "tracks": [
              {
                "name": "The Wicker Man"
              }
            ]
          },
          {
            "title": "Dance Of Death",
            "tracks": [
              {
                "name": "Wildest Dreams"
              }
            ]
          },
          {
            "title": "Fear Of The Dark",
            "tracks": [
              {
                "name": "Be Quick Or Be Dead"
              }
            ]
          },
          {
            "title": "Iron Maiden",
            "tracks": [
              {
                "name": "01 - Prowler"
              }
            ]
          },
          {
            "title": "Killers",
            "tracks": [
              {
                "name": "The Ides Of March"
              }
            ]
          },
          {
            "title": "Live After Death",
            "tracks": [
              {
                "name": "Intro- Churchill S Speech"
              }
            ]
          },
          {
            "title": "Live At Donington 1992 (Disc 1)",
            "tracks": [
              {
                "name": "Be Quick Or Be Dead"
              }
            ]
          },
          {
            "title": "Live At Donington 1992 (Disc 2)",
            "tracks": [
              {
                "name": "Bring Your Daughter... To The Slaughter..."
              }
            ]
          },
          {
            "title": "No Prayer For The Dying",
            "tracks": [
              {
                "name": "Tailgunner"
              }
            ]
          },
          {
            "title": "Piece Of Mind",
            "tracks": [
              {
                "name": "Where Eagles Dare"
              }
            ]
          },
          {
            "title": "Powerslave",
            "tracks": [
              {
                "name": "Aces High"
              }
            ]
          },
          {
            "title": "Rock In Rio [CD1]",
            "tracks": [
              {
                "name": "Intro"
              }
            ]
          },
          {
            "title": "Rock In Rio [CD2]",
            "tracks": [
              {
                "name": "Dream Of Mirrors"
              }
            ]
          },
          {
            "title": "Seventh Son of a Seventh Son",
            "tracks": [
              {
                "name": "Moonchild"
              }
            ]
          },
          {
            "title": "Somewhere in Time",
            "tracks": [
              {
                "name": "Caught Somewhere in Time"
              }
            ]
          },
          {
            "title": "The Number of The Beast",
            "tracks": [
              {
                "name": "22 Acacia Avenue"
              }
            ]
          },
          {
            "title": "The X Factor",
            "tracks": [
              {
                "name": "Sign Of The Cross"
              }
            ]
          },
          {
            "title": "Virtual XI",
            "tracks": [
              {
                "name": "Futureal"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 91,
        "name": "James Brown",
        "albums": [
          {
            "title": "Sex Machine",
            "tracks": [
              {
                "name": "Please Please Please"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 92,
        "name": "Jamiroquai",
        "albums": [
          {
            "title": "Emergency On Planet Earth",
            "tracks": [
              {
                "name": "When You Gonna Learn (Digeridoo)"
              }
            ]
          },
          {
            "title": "Synkronized",
            "tracks": [
              {
                "name": "Canned Heat"
              }
            ]
          },
          {
            "title": "The Return Of The Space Cowboy",
            "tracks": [
              {
                "name": "Just Another Story"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 93,
        "name": "JET",
        "albums": [
          {
            "title": "Get Born",
            "tracks": [
              {
                "name": "Last Chance"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 94,
        "name": "Jimi Hendrix",
        "albums": [
          {
            "title": "Are You Experienced?",
            "tracks": [
              {
                "name": "Foxy Lady"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 95,
        "name": "Joe Satriani",
        "albums": [
          {
            "title": "Surfing with the Alien (Remastered)",
            "tracks": [
              {
                "name": "Surfing with the Alien"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 96,
        "name": "Jota Quest",
        "albums": [
          {
            "title": "Jota Quest-1995",
            "tracks": [
              {
                "name": "Rapidamente"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 97,
        "name": "João Suplicy",
        "albums": [
          {
            "title": "Cafezinho",
            "tracks": [
              {
                "name": "Pura Elegancia"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 98,
        "name": "Judas Priest",
        "albums": [
          {
            "title": "Living After Midnight",
            "tracks": [
              {
                "name": "The Green Manalishi"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 99,
        "name": "Legião Urbana",
        "albums": [
          {
            "title": "A TempestadeTempestade Ou O Livro Dos Dias",
            "tracks": [
              {
                "name": "Natália"
              }
            ]
          },
          {
            "title": "Mais Do Mesmo",
            "tracks": [
              {
                "name": "Será"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 100,
        "name": "Lenny Kravitz",
        "albums": [
          {
            "title": "Greatest Hits",
            "tracks": [
              {
                "name": "Are You Gonna Go My Way"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 101,
        "name": "Lulu Santos",
        "albums": [
          {
            "title": "Lulu Santos - RCA 100 Anos De Música - Álbum 01",
            "tracks": [
              {
                "name": "Assim Caminha A Humanidade"
              }
            ]
          },
          {
            "title": "Lulu Santos - RCA 100 Anos De Música - Álbum 02",
            "tracks": [
              {
                "name": "Honolulu"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 102,
        "name": "Marillion",
        "albums": [
          {
            "title": "Misplaced Childhood",
            "tracks": [
              {
                "name": "Pseudo Silk Kimono"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 103,
        "name": "Marisa Monte",
        "albums": [
          {
            "title": "Barulhinho Bom",
            "tracks": [
              {
                "name": "Arrepio"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 104,
        "name": "Marvin Gaye",
        "albums": [
          {
            "title": "Seek And Shall Find: More Of The Best (1963-1981)",
            "tracks": [
              {
                "name": "Wherever I Lay My Hat"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 105,
        "name": "Men At Work",
        "albums": [
          {
            "title": "The Best Of Men At Work",
            "tracks": [
              {
                "name": "Down Under"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 106,
        "name": "Motörhead",
        "albums": [
          {
            "title": "Ace Of Spades",
            "tracks": [
              {
                "name": "Ace Of Spades"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 107,
        "name": "Motörhead & Girlschool",
        "albums": []
      },
      {
        "artist_id": 108,
        "name": "Mônica Marianno",
        "albums": [
          {
            "title": "Demorou...",
            "tracks": [
              {
                "name": "Kir Royal"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 109,
        "name": "Mötley Crüe",
        "albums": [
          {
            "title": "Motley Crue Greatest Hits",
            "tracks": [
              {
                "name": "Bitter Pill"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 110,
        "name": "Nirvana",
        "albums": [
          {
            "title": "From The Muddy Banks Of The Wishkah [Live]",
            "tracks": [
              {
                "name": "Intro"
              }
            ]
          },
          {
            "title": "Nevermind",
            "tracks": [
              {
                "name": "Smells Like Teen Spirit"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 111,
        "name": "O Terço",
        "albums": [
          {
            "title": "Compositores",
            "tracks": [
              {
                "name": "Time"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 112,
        "name": "Olodum",
        "albums": [
          {
            "title": "Olodum",
            "tracks": [
              {
                "name": "Requebra"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 113,
        "name": "Os Paralamas Do Sucesso",
        "albums": [
          {
            "title": "Acústico MTV",
            "tracks": [
              {
                "name": "Vulcão Dub - Fui Eu"
              }
            ]
          },
          {
            "title": "Arquivo II",
            "tracks": [
              {
                "name": "Trac Trac"
              }
            ]
          },
          {
            "title": "Arquivo Os Paralamas Do Sucesso",
            "tracks": [
              {
                "name": "Caleidoscópio"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 114,
        "name": "Ozzy Osbourne",
        "albums": [
          {
            "title": "Bark at the Moon (Remastered)",
            "tracks": [
              {
                "name": "Bark at the Moon"
              }
            ]
          },
          {
            "title": "Blizzard of Ozz",
            "tracks": [
              {
                "name": "I Don't Know"
              }
            ]
          },
          {
            "title": "Diary of a Madman (Remastered)",
            "tracks": [
              {
                "name": "Flying High Again"
              }
            ]
          },
          {
            "title": "No More Tears (Remastered)",
            "tracks": [
              {
                "name": "Mama, I'm Coming Home"
              }
            ]
          },
          {
            "title": "Tribute",
            "tracks": [
              {
                "name": "I Don't Know"
              }
            ]
          },
          {
            "title": "Speak of the Devil",
            "tracks": [
              {
                "name": "Sympton of the Universe"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 115,
        "name": "Page & Plant",
        "albums": [
          {
            "title": "Walking Into Clarksdale",
            "tracks": [
              {
                "name": "Shining In The Light"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 116,
        "name": "Passengers",
        "albums": [
          {
            "title": "Original Soundtracks 1",
            "tracks": [
              {
                "name": "United Colours"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 117,
        "name": "Paul D'Ianno",
        "albums": [
          {
            "title": "The Beast Live",
            "tracks": [
              {
                "name": "Wrathchild"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 118,
        "name": "Pearl Jam",
        "albums": [
          {
            "title": "Live On Two Legs [Live]",
            "tracks": [
              {
                "name": "Corduroy"
              }
            ]
          },
          {
            "title": "Pearl Jam",
            "tracks": [
              {
                "name": "Life Wasted"
              }
            ]
          },
          {
            "title": "Riot Act",
            "tracks": [
              {
                "name": "Can't Keep"
              }
            ]
          },
          {
            "title": "Ten",
            "tracks": [
              {
                "name": "Once"
              }
            ]
          },
          {
            "title": "Vs.",
            "tracks": [
              {
                "name": "Go"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 119,
        "name": "Peter Tosh",
        "albums": []
      },
      {
        "artist_id": 120,
        "name": "Pink Floyd",
        "albums": [
          {
            "title": "Dark Side Of The Moon",
            "tracks": [
              {
                "name": "Speak To Me/Breathe"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 121,
        "name": "Planet Hemp",
        "albums": [
          {
            "title": "Os Cães Ladram Mas A Caravana Não Pára",
            "tracks": [
              {
                "name": "ZeroVinteUm"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 122,
        "name": "R.E.M. Feat. Kate Pearson",
        "albums": [
          {
            "title": "Out Of Time",
            "tracks": [
              {
                "name": "Shiny Happy People"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 123,
        "name": "R.E.M. Feat. KRS-One",
        "albums": []
      },
      {
        "artist_id": 124,
        "name": "R.E.M.",
        "albums": [
          {
            "title": "Green",
            "tracks": [
              {
                "name": "Pop Song 89"
              }
            ]
          },
          {
            "title": "New Adventures In Hi-Fi",
            "tracks": [
              {
                "name": "How The West Was Won And Where It Got Us"
              }
            ]
          },
          {
            "title": "The Best Of R.E.M.: The IRS Years",
            "tracks": [
              {
                "name": "Carnival Of Sorts"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 125,
        "name": "Raimundos",
        "albums": [
          {
            "title": "Cesta Básica",
            "tracks": [
              {
                "name": "Infeliz Natal"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 126,
        "name": "Raul Seixas",
        "albums": [
          {
            "title": "Raul Seixas",
            "tracks": [
              {
                "name": "Maluco Beleza"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 127,
        "name": "Red Hot Chili Peppers",
        "albums": [
          {
            "title": "Blood Sugar Sex Magik",
            "tracks": [
              {
                "name": "The Power Of Equality"
              }
            ]
          },
          {
            "title": "By The Way",
            "tracks": [
              {
                "name": "By The Way"
              }
            ]
          },
          {
            "title": "Californication",
            "tracks": [
              {
                "name": "Around The World"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 128,
        "name": "Rush",
        "albums": [
          {
            "title": "Retrospective I (1974-1980)",
            "tracks": [
              {
                "name": "The Spirit Of Radio"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 129,
        "name": "Simply Red",
        "albums": []
      },
      {
        "artist_id": 130,
        "name": "Skank",
        "albums": [
          {
            "title": "Maquinarama",
            "tracks": [
              {
                "name": "Água E Fogo"
              }
            ]
          },
          {
            "title": "O Samba Poconé",
            "tracks": [
              {
                "name": "É Uma Partida De Futebol"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 131,
        "name": "Smashing Pumpkins",
        "albums": [
          {
            "title": "Judas 0: B-Sides and Rarities",
            "tracks": [
              {
                "name": "Lucky 13"
              }
            ]
          },
          {
            "title": "Rotten Apples: Greatest Hits",
            "tracks": [
              {
                "name": "Siva"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 132,
        "name": "Soundgarden",
        "albums": [
          {
            "title": "A-Sides",
            "tracks": [
              {
                "name": "Nothing To Say"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 133,
        "name": "Stevie Ray Vaughan & Double Trouble",
        "albums": [
          {
            "title": "In Step",
            "tracks": [
              {
                "name": "The House Is Rockin'"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 134,
        "name": "Stone Temple Pilots",
        "albums": [
          {
            "title": "Core",
            "tracks": [
              {
                "name": "Dead And Bloated"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 135,
        "name": "System Of A Down",
        "albums": [
          {
            "title": "Mezmerize",
            "tracks": [
              {
                "name": "Soldier Side - Intro"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 136,
        "name": "Terry Bozzio, Tony Levin & Steve Stevens",
        "albums": [
          {
            "title": "[1997] Black Light Syndrome",
            "tracks": [
              {
                "name": "The Sun Road"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 137,
        "name": "The Black Crowes",
        "albums": [
          {
            "title": "Live [Disc 1]",
            "tracks": [
              {
                "name": "Midnight From The Inside Out"
              }
            ]
          },
          {
            "title": "Live [Disc 2]",
            "tracks": [
              {
                "name": "Black Moon Creeping"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 138,
        "name": "The Clash",
        "albums": [
          {
            "title": "The Singles",
            "tracks": [
              {
                "name": "White Riot"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 139,
        "name": "The Cult",
        "albums": [
          {
            "title": "Beyond Good And Evil",
            "tracks": [
              {
                "name": "War (The Process)"
              }
            ]
          },
          {
            "title": "Pure Cult: The Best Of The Cult (For Rockers, Ravers, Lovers & Sinners) [UK]",
            "tracks": [
              {
                "name": "She Sells Sanctuary"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 140,
        "name": "The Doors",
        "albums": [
          {
            "title": "The Doors",
            "tracks": [
              {
                "name": "Break on Through"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 141,
        "name": "The Police",
        "albums": [
          {
            "title": "The Police Greatest Hits",
            "tracks": [
              {
                "name": "Roxanne"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 142,
        "name": "The Rolling Stones",
        "albums": [
          {
            "title": "Hot Rocks, 1964-1971 (Disc 1)",
            "tracks": [
              {
                "name": "Time Is On My Side"
              }
            ]
          },
          {
            "title": "No Security",
            "tracks": [
              {
                "name": "Intro"
              }
            ]
          },
          {
            "title": "Voodoo Lounge",
            "tracks": [
              {
                "name": "Love Is Strong"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 143,
        "name": "The Tea Party",
        "albums": [
          {
            "title": "Tangents",
            "tracks": [
              {
                "name": "Walking Wounded"
              }
            ]
          },
          {
            "title": "Transmission",
            "tracks": [
              {
                "name": "Temptation"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 144,
        "name": "The Who",
        "albums": [
          {
            "title": "My Generation - The Very Best Of The Who",
            "tracks": [
              {
                "name": "I Can't Explain"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 145,
        "name": "Tim Maia",
        "albums": [
          {
            "title": "Serie Sem Limite (Disc 1)",
            "tracks": [
              {
                "name": "Primavera"
              }
            ]
          },
          {
            "title": "Serie Sem Limite (Disc 2)",
            "tracks": [
              {
                "name": "O Que Me Importa"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 146,
        "name": "Titãs",
        "albums": [
          {
            "title": "Acústico",
            "tracks": [
              {
                "name": "Comida"
              }
            ]
          },
          {
            "title": "Volume Dois",
            "tracks": [
              {
                "name": "Sonifera Ilha"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 147,
        "name": "Battlestar Galactica",
        "albums": [
          {
            "title": "Battlestar Galactica: The Story So Far",
            "tracks": [
              {
                "name": "Battlestar Galactica: The Story So Far"
              }
            ]
          },
          {
            "title": "Battlestar Galactica, Season 3",
            "tracks": [
              {
                "name": "Occupation / Precipice"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 148,
        "name": "Heroes",
        "albums": [
          {
            "title": "Heroes, Season 1",
            "tracks": [
              {
                "name": "Genesis"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 149,
        "name": "Lost",
        "albums": [
          {
            "title": "Lost, Season 3",
            "tracks": [
              {
                "name": "A Tale of Two Cities"
              }
            ]
          },
          {
            "title": "Lost, Season 1",
            "tracks": [
              {
                "name": "Lost (Pilot, Part 1) [Premiere]"
              }
            ]
          },
          {
            "title": "Lost, Season 2",
            "tracks": [
              {
                "name": "Man of Science, Man of Faith (Premiere)"
              }
            ]
          },
          {
            "title": "LOST, Season 4",
            "tracks": [
              {
                "name": "Past, Present, and Future"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 150,
        "name": "U2",
        "albums": [
          {
            "title": "Achtung Baby",
            "tracks": [
              {
                "name": "Zoo Station"
              }
            ]
          },
          {
            "title": "All That You Can't Leave Behind",
            "tracks": [
              {
                "name": "Beautiful Day"
              }
            ]
          },
          {
            "title": "B-Sides 1980-1990",
            "tracks": [
              {
                "name": "The Three Sunrises"
              }
            ]
          },
          {
            "title": "How To Dismantle An Atomic Bomb",
            "tracks": [
              {
                "name": "Vertigo"
              }
            ]
          },
          {
            "title": "Pop",
            "tracks": [
              {
                "name": "Discotheque"
              }
            ]
          },
          {
            "title": "Rattle And Hum",
            "tracks": [
              {
                "name": "Helter Skelter"
              }
            ]
          },
          {
            "title": "The Best Of 1980-1990",
            "tracks": [
              {
                "name": "Pride (In The Name Of Love)"
              }
            ]
          },
          {
            "title": "War",
            "tracks": [
              {
                "name": "Sunday Bloody Sunday"
              }
            ]
          },
          {
            "title": "Zooropa",
            "tracks": [
              {
                "name": "Zooropa"
              }
            ]
          },
          {
            "title": "Instant Karma: The Amnesty International Campaign to Save Darfur",
            "tracks": [
              {
                "name": "Instant Karma"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 151,
        "name": "UB40",
        "albums": [
          {
            "title": "UB40 The Best Of - Volume Two [UK]",
            "tracks": [
              {
                "name": "Breakfast In Bed"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 152,
        "name": "Van Halen",
        "albums": [
          {
            "title": "Diver Down",
            "tracks": [
              {
                "name": "Where Have All The Good Times Gone?"
              }
            ]
          },
          {
            "title": "The Best Of Van Halen, Vol. I",
            "tracks": [
              {
                "name": "Eruption"
              }
            ]
          },
          {
            "title": "Van Halen",
            "tracks": [
              {
                "name": "Runnin' With The Devil"
              }
            ]
          },
          {
            "title": "Van Halen III",
            "tracks": [
              {
                "name": "Neworld"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 153,
        "name": "Velvet Revolver",
        "albums": [
          {
            "title": "Contraband",
            "tracks": [
              {
                "name": "Sucker Train Blues"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 154,
        "name": "Whitesnake",
        "albums": []
      },
      {
        "artist_id": 155,
        "name": "Zeca Pagodinho",
        "albums": [
          {
            "title": "Ao Vivo [IMPORT]",
            "tracks": [
              {
                "name": "Faixa Amarela"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 156,
        "name": "The Office",
        "albums": [
          {
            "title": "The Office, Season 1",
            "tracks": [
              {
                "name": "The Office: An American Workplace (Pilot)"
              }
            ]
          },
          {
            "title": "The Office, Season 2",
            "tracks": [
              {
                "name": "The Dundies"
              }
            ]
          },
          {
            "title": "The Office, Season 3",
            "tracks": [
              {
                "name": "Gay Witch Hunt"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 157,
        "name": "Dread Zeppelin",
        "albums": [
          {
            "title": "Un-Led-Ed",
            "tracks": [
              {
                "name": "Your Time Is Gonna Come"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 158,
        "name": "Battlestar Galactica (Classic)",
        "albums": [
          {
            "title": "Battlestar Galactica (Classic), Season 1",
            "tracks": [
              {
                "name": "Battlestar Galactica, Pt. 1"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 159,
        "name": "Aquaman",
        "albums": [
          {
            "title": "Aquaman",
            "tracks": [
              {
                "name": "Pilot"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 160,
        "name": "Christina Aguilera featuring BigElf",
        "albums": []
      },
      {
        "artist_id": 161,
        "name": "Aerosmith & Sierra Leone's Refugee Allstars",
        "albums": []
      },
      {
        "artist_id": 162,
        "name": "Los Lonely Boys",
        "albums": []
      },
      {
        "artist_id": 163,
        "name": "Corinne Bailey Rae",
        "albums": []
      },
      {
        "artist_id": 164,
        "name": "Dhani Harrison & Jakob Dylan",
        "albums": []
      },
      {
        "artist_id": 165,
        "name": "Jackson Browne",
        "albums": []
      },
      {
        "artist_id": 166,
        "name": "Avril Lavigne",
        "albums": []
      },
      {
        "artist_id": 167,
        "name": "Big & Rich",
        "albums": []
      },
      {
        "artist_id": 168,
        "name": "Youssou N'Dour",
        "albums": []
      },
      {
        "artist_id": 169,
        "name": "Black Eyed Peas",
        "albums": []
      },
      {
        "artist_id": 170,
        "name": "Jack Johnson",
        "albums": []
      },
      {
        "artist_id": 171,
        "name": "Ben Harper",
        "albums": []
      },
      {
        "artist_id": 172,
        "name": "Snow Patrol",
        "albums": []
      },
      {
        "artist_id": 173,
        "name": "Matisyahu",
        "albums": []
      },
      {
        "artist_id": 174,
        "name": "The Postal Service",
        "albums": []
      },
      {
        "artist_id": 175,
        "name": "Jaguares",
        "albums": []
      },
      {
        "artist_id": 176,
        "name": "The Flaming Lips",
        "albums": []
      },
      {
        "artist_id": 177,
        "name": "Jack's Mannequin & Mick Fleetwood",
        "albums": []
      },
      {
        "artist_id": 178,
        "name": "Regina Spektor",
        "albums": []
      },
      {
        "artist_id": 179,
        "name": "Scorpions",
        "albums": [
          {
            "title": "20th Century Masters - The Millennium Collection: The Best of Scorpions",
            "tracks": [
              {
                "name": "Rock You Like a Hurricane"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 180,
        "name": "House Of Pain",
        "albums": [
          {
            "title": "House of Pain",
            "tracks": [
              {
                "name": "Jump Around"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 181,
        "name": "Xis",
        "albums": []
      },
      {
        "artist_id": 182,
        "name": "Nega Gizza",
        "albums": []
      },
      {
        "artist_id": 183,
        "name": "Gustavo & Andres Veiga & Salazar",
        "albums": []
      },
      {
        "artist_id": 184,
        "name": "Rodox",
        "albums": []
      },
      {
        "artist_id": 185,
        "name": "Charlie Brown Jr.",
        "albums": []
      },
      {
        "artist_id": 186,
        "name": "Pedro Luís E A Parede",
        "albums": []
      },
      {
        "artist_id": 187,
        "name": "Los Hermanos",
        "albums": []
      },
      {
        "artist_id": 188,
        "name": "Mundo Livre S/A",
        "albums": []
      },
      {
        "artist_id": 189,
        "name": "Otto",
        "albums": []
      },
      {
        "artist_id": 190,
        "name": "Instituto",
        "albums": []
      },
      {
        "artist_id": 191,
        "name": "Nação Zumbi",
        "albums": []
      },
      {
        "artist_id": 192,
        "name": "DJ Dolores & Orchestra Santa Massa",
        "albums": []
      },
      {
        "artist_id": 193,
        "name": "Seu Jorge",
        "albums": []
      },
      {
        "artist_id": 194,
        "name": "Sabotage E Instituto",
        "albums": []
      },
      {
        "artist_id": 195,
        "name": "Stereo Maracana",
        "albums": []
      },
      {
        "artist_id": 196,
        "name": "Cake",
        "albums": [
          {
            "title": "Cake: B-Sides and Rarities",
            "tracks": [
              {
                "name": "War Pigs"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 197,
        "name": "Aisha Duo",
        "albums": [
          {
            "title": "Quiet Songs",
            "tracks": [
              {
                "name": "Amanda"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 198,
        "name": "Habib Koité and Bamada",
        "albums": [
          {
            "title": "Muso Ko",
            "tracks": [
              {
                "name": "Din Din Wo (Little Child)"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 199,
        "name": "Karsh Kale",
        "albums": [
          {
            "title": "Realize",
            "tracks": [
              {
                "name": "Distance"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 200,
        "name": "The Posies",
        "albums": [
          {
            "title": "Every Kind of Light",
            "tracks": [
              {
                "name": "I Guess You're Right"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 201,
        "name": "Luciana Souza/Romero Lubambo",
        "albums": [
          {
            "title": "Duos II",
            "tracks": [
              {
                "name": "Muita Bobeira"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 202,
        "name": "Aaron Goldberg",
        "albums": [
          {
            "title": "Worlds",
            "tracks": [
              {
                "name": "OAM's Blues"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 203,
        "name": "Nicolaus Esterhazy Sinfonia",
        "albums": [
          {
            "title": "The Best of Beethoven",
            "tracks": [
              {
                "name": "Symphony No. 3 in E-flat major, Op. 55, \"Eroica\" - Scherzo: Allegro Vivace"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 204,
        "name": "Temple of the Dog",
        "albums": [
          {
            "title": "Temple of the Dog",
            "tracks": [
              {
                "name": "Say Hello 2 Heaven"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 205,
        "name": "Chris Cornell",
        "albums": [
          {
            "title": "Carry On",
            "tracks": [
              {
                "name": "No Such Thing"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 206,
        "name": "Alberto Turco & Nova Schola Gregoriana",
        "albums": [
          {
            "title": "Adorate Deum: Gregorian Chant from the Proper of the Mass",
            "tracks": [
              {
                "name": "Intoitus: Adorate Deum"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 207,
        "name": "Richard Marlow & The Choir of Trinity College, Cambridge",
        "albums": [
          {
            "title": "Allegri: Miserere",
            "tracks": [
              {
                "name": "Miserere mei, Deus"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 208,
        "name": "English Concert & Trevor Pinnock",
        "albums": [
          {
            "title": "Pachelbel: Canon & Gigue",
            "tracks": [
              {
                "name": "Canon and Gigue in D Major: I. Canon"
              }
            ]
          },
          {
            "title": "Handel: Music for the Royal Fireworks (Original Version 1749)",
            "tracks": [
              {
                "name": "Music for the Royal Fireworks, HWV351 (1749): La Réjouissance"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 209,
        "name": "Anne-Sophie Mutter, Herbert Von Karajan & Wiener Philharmoniker",
        "albums": [
          {
            "title": "Vivaldi: The Four Seasons",
            "tracks": [
              {
                "name": "Concerto No. 1 in E Major, RV 269 \"Spring\": I. Allegro"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 210,
        "name": "Hilary Hahn, Jeffrey Kahane, Los Angeles Chamber Orchestra & Margaret Batjer",
        "albums": [
          {
            "title": "Bach: Violin Concertos",
            "tracks": [
              {
                "name": "Concerto for 2 Violins in D Minor, BWV 1043: I. Vivace"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 211,
        "name": "Wilhelm Kempff",
        "albums": [
          {
            "title": "Bach: Goldberg Variations",
            "tracks": [
              {
                "name": "Aria Mit 30 Veränderungen, BWV 988 \"Goldberg Variations\": Aria"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 212,
        "name": "Yo-Yo Ma",
        "albums": [
          {
            "title": "Bach: The Cello Suites",
            "tracks": [
              {
                "name": "Suite for Solo Cello No. 1 in G Major, BWV 1007: I. Prélude"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 213,
        "name": "Scholars Baroque Ensemble",
        "albums": [
          {
            "title": "Handel: The Messiah (Highlights)",
            "tracks": [
              {
                "name": "The Messiah: Behold, I Tell You a Mystery... The Trumpet Shall Sound"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 214,
        "name": "Academy of St. Martin in the Fields & Sir Neville Marriner",
        "albums": [
          {
            "title": "The World of Classical Favourites",
            "tracks": [
              {
                "name": "Solomon HWV 67: The Arrival of the Queen of Sheba"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 215,
        "name": "Academy of St. Martin in the Fields Chamber Ensemble & Sir Neville Marriner",
        "albums": [
          {
            "title": "Sir Neville Marriner: A Celebration",
            "tracks": [
              {
                "name": "\"Eine Kleine Nachtmusik\" Serenade In G, K. 525: I. Allegro"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 216,
        "name": "Berliner Philharmoniker, Claudio Abbado & Sabine Meyer",
        "albums": [
          {
            "title": "Mozart: Wind Concertos",
            "tracks": [
              {
                "name": "Concerto for Clarinet in A Major, K. 622: II. Adagio"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 217,
        "name": "Royal Philharmonic Orchestra & Sir Thomas Beecham",
        "albums": [
          {
            "title": "Haydn: Symphonies 99 - 104",
            "tracks": [
              {
                "name": "Symphony No. 104 in D Major \"London\": IV. Finale: Spiritoso"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 218,
        "name": "Orchestre Révolutionnaire et Romantique & John Eliot Gardiner",
        "albums": [
          {
            "title": "Beethoven: Symhonies Nos. 5 & 6",
            "tracks": [
              {
                "name": "Symphony No.5 in C Minor: I. Allegro con brio"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 219,
        "name": "Britten Sinfonia, Ivor Bolton & Lesley Garrett",
        "albums": [
          {
            "title": "A Soprano Inspired",
            "tracks": [
              {
                "name": "Ave Maria"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 220,
        "name": "Chicago Symphony Chorus, Chicago Symphony Orchestra & Sir Georg Solti",
        "albums": [
          {
            "title": "Great Opera Choruses",
            "tracks": [
              {
                "name": "Nabucco: Chorus, \"Va, Pensiero, Sull'ali Dorate\""
              }
            ]
          }
        ]
      },
      {
        "artist_id": 221,
        "name": "Sir Georg Solti & Wiener Philharmoniker",
        "albums": [
          {
            "title": "Wagner: Favourite Overtures",
            "tracks": [
              {
                "name": "Die Walküre: The Ride of the Valkyries"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 222,
        "name": "Academy of St. Martin in the Fields, John Birch, Sir Neville Marriner & Sylvia McNair",
        "albums": [
          {
            "title": "Fauré: Requiem, Ravel: Pavane & Others",
            "tracks": [
              {
                "name": "Requiem, Op.48: 4. Pie Jesu"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 223,
        "name": "London Symphony Orchestra & Sir Charles Mackerras",
        "albums": [
          {
            "title": "Tchaikovsky: The Nutcracker",
            "tracks": [
              {
                "name": "The Nutcracker, Op. 71a, Act II: Scene 14: Pas de deux: Dance of the Prince & the Sugar-Plum Fairy"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 224,
        "name": "Barry Wordsworth & BBC Concert Orchestra",
        "albums": [
          {
            "title": "The Last Night of the Proms",
            "tracks": [
              {
                "name": "Nimrod (Adagio) from Variations On an Original Theme, Op. 36 \"Enigma\""
              }
            ]
          }
        ]
      },
      {
        "artist_id": 225,
        "name": "Herbert Von Karajan, Mirella Freni & Wiener Philharmoniker",
        "albums": [
          {
            "title": "Puccini: Madama Butterfly - Highlights",
            "tracks": [
              {
                "name": "Madama Butterfly: Un Bel Dì Vedremo"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 226,
        "name": "Eugene Ormandy",
        "albums": [
          {
            "title": "Holst: The Planets, Op. 32 & Vaughan Williams: Fantasies",
            "tracks": [
              {
                "name": "Jupiter, the Bringer of Jollity"
              }
            ]
          },
          {
            "title": "Strauss: Waltzes",
            "tracks": [
              {
                "name": "On the Beautiful Blue Danube"
              }
            ]
          },
          {
            "title": "Respighi:Pines of Rome",
            "tracks": [
              {
                "name": "Pini Di Roma (Pinien Von Rom) \\ I Pini Della Via Appia"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 227,
        "name": "Luciano Pavarotti",
        "albums": [
          {
            "title": "Pavarotti's Opera Made Easy",
            "tracks": [
              {
                "name": "Turandot, Act III, Nessun dorma!"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 228,
        "name": "Leonard Bernstein & New York Philharmonic",
        "albums": [
          {
            "title": "Great Performances - Barber's Adagio and Other Romantic Favorites for Strings",
            "tracks": [
              {
                "name": "Adagio for Strings from the String Quartet, Op. 11"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 229,
        "name": "Boston Symphony Orchestra & Seiji Ozawa",
        "albums": [
          {
            "title": "Carmina Burana",
            "tracks": [
              {
                "name": "Carmina Burana: O Fortuna"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 230,
        "name": "Aaron Copland & London Symphony Orchestra",
        "albums": [
          {
            "title": "A Copland Celebration, Vol. I",
            "tracks": [
              {
                "name": "Fanfare for the Common Man"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 231,
        "name": "Ton Koopman",
        "albums": [
          {
            "title": "Bach: Toccata & Fugue in D Minor",
            "tracks": [
              {
                "name": "Toccata and Fugue in D Minor, BWV 565: I. Toccata"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 232,
        "name": "Sergei Prokofiev & Yuri Temirkanov",
        "albums": [
          {
            "title": "Prokofiev: Symphony No.1",
            "tracks": [
              {
                "name": "Symphony No.1 in D Major, Op.25 \"Classical\", Allegro Con Brio"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 233,
        "name": "Chicago Symphony Orchestra & Fritz Reiner",
        "albums": [
          {
            "title": "Scheherazade",
            "tracks": [
              {
                "name": "Scheherazade, Op. 35: I. The Sea and Sindbad's Ship"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 234,
        "name": "Orchestra of The Age of Enlightenment",
        "albums": [
          {
            "title": "Bach: The Brandenburg Concertos",
            "tracks": [
              {
                "name": "Concerto No.2 in F Major, BWV1047, I. Allegro"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 235,
        "name": "Emanuel Ax, Eugene Ormandy & Philadelphia Orchestra",
        "albums": [
          {
            "title": "Chopin: Piano Concertos Nos. 1 & 2",
            "tracks": [
              {
                "name": "Concerto for Piano No. 2 in F Minor, Op. 21: II. Larghetto"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 236,
        "name": "James Levine",
        "albums": [
          {
            "title": "Mascagni: Cavalleria Rusticana",
            "tracks": [
              {
                "name": "Cavalleria Rusticana \\ Act \\ Intermezzo Sinfonico"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 237,
        "name": "Berliner Philharmoniker & Hans Rosbaud",
        "albums": [
          {
            "title": "Sibelius: Finlandia",
            "tracks": [
              {
                "name": "Karelia Suite, Op.11: 2. Ballade (Tempo Di Menuetto)"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 238,
        "name": "Maurizio Pollini",
        "albums": [
          {
            "title": "Beethoven Piano Sonatas: Moonlight & Pastorale",
            "tracks": [
              {
                "name": "Piano Sonata No. 14 in C Sharp Minor, Op. 27, No. 2, \"Moonlight\": I. Adagio sostenuto"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 239,
        "name": "Academy of St. Martin in the Fields, Sir Neville Marriner & William Bennett",
        "albums": []
      },
      {
        "artist_id": 240,
        "name": "Gustav Mahler",
        "albums": [
          {
            "title": "Great Recordings of the Century - Mahler: Das Lied von der Erde",
            "tracks": [
              {
                "name": "Das Lied Von Der Erde, Von Der Jugend"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 241,
        "name": "Felix Schmidt, London Symphony Orchestra & Rafael Frühbeck de Burgos",
        "albums": [
          {
            "title": "Elgar: Cello Concerto & Vaughan Williams: Fantasias",
            "tracks": [
              {
                "name": "Concerto for Cello and Orchestra in E minor, Op. 85: I. Adagio - Moderato"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 242,
        "name": "Edo de Waart & San Francisco Symphony",
        "albums": [
          {
            "title": "Adams, John: The Chairman Dances",
            "tracks": [
              {
                "name": "Two Fanfares for Orchestra: II. Short Ride in a Fast Machine"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 243,
        "name": "Antal Doráti & London Symphony Orchestra",
        "albums": [
          {
            "title": "Tchaikovsky: 1812 Festival Overture, Op.49, Capriccio Italien & Beethoven: Wellington's Victory",
            "tracks": [
              {
                "name": "Wellington's Victory or the Battle Symphony, Op.91: 2. Symphony of Triumph"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 244,
        "name": "Choir Of Westminster Abbey & Simon Preston",
        "albums": [
          {
            "title": "Palestrina: Missa Papae Marcelli & Allegri: Miserere",
            "tracks": [
              {
                "name": "Missa Papae Marcelli: Kyrie"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 245,
        "name": "Michael Tilson Thomas & San Francisco Symphony",
        "albums": [
          {
            "title": "Prokofiev: Romeo & Juliet",
            "tracks": [
              {
                "name": "Romeo et Juliette: No. 11 - Danse des Chevaliers"
              }
            ]
          },
          {
            "title": "Berlioz: Symphonie Fantastique",
            "tracks": [
              {
                "name": "Symphonie Fantastique, Op. 14: V. Songe d'une nuit du sabbat"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 246,
        "name": "Chor der Wiener Staatsoper, Herbert Von Karajan & Wiener Philharmoniker",
        "albums": [
          {
            "title": "Bizet: Carmen Highlights",
            "tracks": [
              {
                "name": "Carmen: Overture"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 247,
        "name": "The King's Singers",
        "albums": [
          {
            "title": "English Renaissance",
            "tracks": [
              {
                "name": "Lamentations of Jeremiah, First Set \\ Incipit Lamentatio"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 248,
        "name": "Berliner Philharmoniker & Herbert Von Karajan",
        "albums": [
          {
            "title": "Grieg: Peer Gynt Suites & Sibelius: Pelléas et Mélisande",
            "tracks": [
              {
                "name": "Peer Gynt Suite No.1, Op.46: 1. Morning Mood"
              }
            ]
          },
          {
            "title": "Mozart: Symphonies Nos. 40 & 41",
            "tracks": [
              {
                "name": "Symphony No. 41 in C Major, K. 551, \"Jupiter\": IV. Molto allegro"
              }
            ]
          },
          {
            "title": "Prokofiev: Symphony No.5 & Stravinksy: Le Sacre Du Printemps",
            "tracks": [
              {
                "name": "Le Sacre Du Printemps: I.iv. Spring Rounds"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 249,
        "name": "Sir Georg Solti, Sumi Jo & Wiener Philharmoniker",
        "albums": [
          {
            "title": "Mozart Gala: Famous Arias",
            "tracks": [
              {
                "name": "Die Zauberflöte, K.620: \"Der Hölle Rache Kocht in Meinem Herze\""
              }
            ]
          }
        ]
      },
      {
        "artist_id": 250,
        "name": "Christopher O'Riley",
        "albums": [
          {
            "title": "SCRIABIN: Vers la flamme",
            "tracks": [
              {
                "name": "SCRIABIN: Prelude in B Major, Op. 11, No. 11"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 251,
        "name": "Fretwork",
        "albums": [
          {
            "title": "Armada: Music from the Courts of England and Spain",
            "tracks": [
              {
                "name": "Pavan, Lachrimae Antiquae"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 252,
        "name": "Amy Winehouse",
        "albums": [
          {
            "title": "Back to Black",
            "tracks": [
              {
                "name": "Rehab"
              }
            ]
          },
          {
            "title": "Frank",
            "tracks": [
              {
                "name": "Intro / Stronger Than Me"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 253,
        "name": "Calexico",
        "albums": [
          {
            "title": "Carried to Dust (Bonus Track Version)",
            "tracks": [
              {
                "name": "Slowness"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 254,
        "name": "Otto Klemperer & Philharmonia Orchestra",
        "albums": [
          {
            "title": "Beethoven: Symphony No. 6 'Pastoral' Etc.",
            "tracks": [
              {
                "name": "Prometheus Overture, Op. 43"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 255,
        "name": "Yehudi Menuhin",
        "albums": [
          {
            "title": "Bartok: Violin & Viola Concertos",
            "tracks": [
              {
                "name": "Sonata for Solo Violin: IV: Presto"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 256,
        "name": "Philharmonia Orchestra & Sir Neville Marriner",
        "albums": [
          {
            "title": "Mendelssohn: A Midsummer Night's Dream",
            "tracks": [
              {
                "name": "A Midsummer Night's Dream, Op.61 Incidental Music: No.7 Notturno"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 257,
        "name": "Academy of St. Martin in the Fields, Sir Neville Marriner & Thurston Dart",
        "albums": [
          {
            "title": "Bach: Orchestral Suites Nos. 1 - 4",
            "tracks": [
              {
                "name": "Suite No. 3 in D, BWV 1068: III. Gavotte I & II"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 258,
        "name": "Les Arts Florissants & William Christie",
        "albums": [
          {
            "title": "Charpentier: Divertissements, Airs & Concerts",
            "tracks": [
              {
                "name": "Concert pour 4 Parties de V**les, H. 545: I. Prelude"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 259,
        "name": "The 12 Cellists of The Berlin Philharmonic",
        "albums": [
          {
            "title": "South American Getaway",
            "tracks": [
              {
                "name": "Adios nonino"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 260,
        "name": "Adrian Leaper & Doreen de Feis",
        "albums": [
          {
            "title": "Górecki: Symphony No. 3",
            "tracks": [
              {
                "name": "Symphony No. 3 Op. 36 for Orchestra and Soprano \"Symfonia Piesni Zalosnych\" \\ Lento E Largo - Tranquillissimo"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 261,
        "name": "Roger Norrington, London Classical Players",
        "albums": [
          {
            "title": "Purcell: The Fairy Queen",
            "tracks": [
              {
                "name": "Act IV, Symphony"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 262,
        "name": "Charles Dutoit & L'Orchestre Symphonique de Montréal",
        "albums": [
          {
            "title": "The Ultimate Relexation Album",
            "tracks": [
              {
                "name": "3 Gymnopédies: No.1 - Lent Et Grave, No.3 - Lent Et Douloureux"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 263,
        "name": "Equale Brass Ensemble, John Eliot Gardiner & Munich Monteverdi Orchestra and Choir",
        "albums": [
          {
            "title": "Purcell: Music for the Queen Mary",
            "tracks": [
              {
                "name": "Music for the Funeral of Queen Mary: VI. \"Thou Knowest, Lord, the Secrets of Our Hearts\""
              }
            ]
          }
        ]
      },
      {
        "artist_id": 264,
        "name": "Kent Nagano and Orchestre de l'Opéra de Lyon",
        "albums": [
          {
            "title": "Weill: The Seven Deadly Sins",
            "tracks": [
              {
                "name": "Symphony No. 2: III. Allegro vivace"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 265,
        "name": "Julian Bream",
        "albums": [
          {
            "title": "J.S. Bach: Chaconne, Suite in E Minor, Partita in E Major & Prelude, Fugue and Allegro",
            "tracks": [
              {
                "name": "Partita in E Major, BWV 1006A: I. Prelude"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 266,
        "name": "Martin Roscoe",
        "albums": [
          {
            "title": "Szymanowski: Piano Works, Vol. 1",
            "tracks": [
              {
                "name": "Metopes, Op. 29: Calypso"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 267,
        "name": "Göteborgs Symfoniker & Neeme Järvi",
        "albums": [
          {
            "title": "Nielsen: The Six Symphonies",
            "tracks": [
              {
                "name": "Symphony No. 2, Op. 16 -  \"The Four Temperaments\": II. Allegro Comodo e Flemmatico"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 268,
        "name": "Itzhak Perlman",
        "albums": [
          {
            "title": "Great Recordings of the Century: Paganini's 24 Caprices",
            "tracks": [
              {
                "name": "24 Caprices, Op. 1, No. 24, for Solo Violin, in A Minor"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 269,
        "name": "Michele Campanella",
        "albums": [
          {
            "title": "Liszt - 12 Études D'Execution Transcendante",
            "tracks": [
              {
                "name": "Étude 1, In C Major - Preludio (Presto) - Liszt"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 270,
        "name": "Gerald Moore",
        "albums": [
          {
            "title": "Great Recordings of the Century - Shubert: Schwanengesang, 4 Lieder",
            "tracks": [
              {
                "name": "Erlkonig, D.328"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 271,
        "name": "Mela Tenenbaum, Pro Musica Prague & Richard Kapp",
        "albums": [
          {
            "title": "Locatelli: Concertos for Violin, Strings and Continuo, Vol. 3",
            "tracks": [
              {
                "name": "Concerto for Violin, Strings and Continuo in G Major, Op. 3, No. 9: I. Allegro"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 272,
        "name": "Emerson String Quartet",
        "albums": [
          {
            "title": "Schubert: The Late String Quartets & String Quintet (3 CD's)",
            "tracks": [
              {
                "name": "String Quartet No. 12 in C Minor, D. 703 \"Quartettsatz\": II. Andante - Allegro assai"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 273,
        "name": "C. Monteverdi, Nigel Rogers - Chiaroscuro; London Baroque; London Cornett & Sackbu",
        "albums": [
          {
            "title": "Monteverdi: L'Orfeo",
            "tracks": [
              {
                "name": "L'orfeo, Act 3, Sinfonia (Orchestra)"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 274,
        "name": "Nash Ensemble",
        "albums": [
          {
            "title": "Mozart: Chamber Music",
            "tracks": [
              {
                "name": "Quintet for Horn, Violin, 2 Violas, and Cello in E Flat Major, K. 407/386c: III. Allegro"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 275,
        "name": "Philip Glass Ensemble",
        "albums": [
          {
            "title": "Koyaanisqatsi (Soundtrack from the Motion Picture)",
            "tracks": [
              {
                "name": "Koyaanisqatsi"
              }
            ]
          }
        ]
      },
      {
        "artist_id": 276,
        "name": "Honey Singh",
        "albums": [
          {
            "title": "Exile on Main St.",
            "tracks": []
          }
        ]
      }
    ]
  }
}
```

3. Get all albums for artist id = 5 without specifying a where clause.

```
chinook=# SET search_path TO public;
SET
chinook=# CREATE VIEW artist_5_albums AS
SELECT * FROM album
WHERE artist_id = 5;
```

```
query {
  artist_5_albums {
    album_id
    title
    artist_id
  }
}
```
```
{
  "data": {
    "artist_5_albums": [
      {
        "album_id": 7,
        "title": "Facelift",
        "artist_id": 5
      }
    ]
  }
}
```
 4. Using a GraphQL mutation, add your favorite artist and one of their albums that isn’t in
the dataset.

```
mutation {
  insert_artist(objects: {artist_id: 276, name: "Honey Singh"}) {
    returning {
      artist_id
      name
    }
  }
}

```

```
{
  "data": {
    "insert_artist": {
      "returning": [
        {
          "artist_id": 276,
          "name": "Honey Singh"
        }
      ]
    }
  }
}
```
```
mutation {
  insert_album(objects: {title: "Exile on Main St.", artist_id: 348}) {
    returning {
      album_id
      title
      artist_id
    }
  }
}
```
```
{
  "data": {
    "insert_album": {
      "returning": [
        {
          "album_id": 348, 
          "title": "Exile on Main St.",
          "artist_id": 101
        }
      ]
    }
  }
}
```

5. How did you identify which ID to include in the mutation?

First, I retrieved the total number of albums and artists using the following GraphQL query:

```
query {
  album_aggregate {
    aggregate {
      count
    }
  }
  artist_aggregate {
    aggregate {
      count
    }
  }
}
```
```
{
  "data": {
    "album_aggregate": {
      "aggregate": {
        "count": 347
      }
    },
    "artist_aggregate": {
      "aggregate": {
        "count": 275
      }
    }
  }
}
```
This query provided the total count of albums and artists in the database.
Then, to add a new artist and album via a mutation, I used the total counts of albums and artists as a reference and incremented them by 1 (since we are adding new data). 

