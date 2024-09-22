# Den hurtige introduktion til Databaser på DAT2

I denne introduktion vil jeg hurtigt gennemgå hvordan vi tidligere har brugt filer til at gemme data, og hvordan vi i fremadrettet gerne vil bruge databaser til at gemme data.

Den gennemgang vores undervisere har lavet, har fokus på en masse nøgleord, som vi skal kunne bruge til eksamen. Jeg vil forsøge både at bruge et mere naturligt sprog og at blande disse ord ind hvor det giver mening.

The code is written for 2. semester on the Datamatiker education in Lyngby.

The code is NOT refactored.

## Gensyn med 1. semester

I ChillFlix projektet læste vi data fra filer og parsede data, sådan at vi kunne oprette nogle objekter, som vi opbevarede i en `List` eller et `Map`.

Det så nogenlunde sådan her ud for film:

```java
    public void parseMovieData() {
        ArrayList<String> movieList = io.readData(moviePath);
        for (String title : movieList) {
            Movie movie = createMovieFromString(title);
            mediaList.put(movie.getTitle(), movie);
            addGenre(movie);
        }
    }
    public Movie createMovieFromString(String movieString) {
        String[] movieData = movieString.split(";");
        String title = movieData[0].trim();
        int releaseYear = Integer.parseInt(movieData[1].trim());
        String genre = movieData[2].trim();
        float rating = Float.parseFloat(movieData[3].replace(",", ".").trim());
        return new Movie(title, releaseYear, genre, rating, 0);
    }
```

Og det var lidt mere kompliceret for serier:

```java
    public void parseSerieData() {
        ArrayList<String> serieList = io.readData(seriePath);
        for (String title : serieList) {
            Serie serie = createSerieFromString(title);
            mediaList.put(serie.getTitle(), serie);
            addGenre(serie);
        }
    }
    public Serie createSerieFromString(String serieString) {
        String[] serieData = serieString.split(";");
        String title = serieData[0].trim();
        String years = serieData[1].trim(); //1991-1992
        int startYear = Integer.parseInt(years.split("-")[0]);
        //int endYear= Integer.parseInt(years.split("-")[1]);
    
        String genre = serieData[2].trim();
        float rating = Float.parseFloat(serieData[3].replace(",", ".").trim());
        String seasonEpisodeString = serieData[4].trim();
        String[] seasonEpisodeList = seasonEpisodeString.split(",");
        Serie serie = new Serie(title, startYear, genre, rating);
        int numberOfSeasons = seasonEpisodeList.length;
        for (String seasonEpisode : seasonEpisodeList) {
            int seasonNumber = Integer.parseInt(seasonEpisode.split("-")[0].trim());
            int episodesInSeason = Integer.parseInt(seasonEpisode.split("-")[1].trim());
            StringBuilder seasonTitle = new StringBuilder(title + " sæson ");
            seasonTitle.append((numberOfSeasons > 10 && seasonNumber < 10) ? "0" : "").append(seasonNumber);
            serie.getSeasonMap().put(seasonTitle.toString(), new TreeMap<>());
            for (int i = 1; i <= episodesInSeason; i++) {
                StringBuilder episodeTitle = new StringBuilder(title);
                episodeTitle.append(" S").append(seasonNumber < 10 ? "0" : "").append(seasonNumber);
                episodeTitle.append("E").append(i < 10 ? "0" : "").append(i);
                Episode ep = new Episode(episodeTitle.toString(), startYear, genre, rating, 0);
                serie.getSeasonMap().get(seasonTitle.toString()).put(ep.getTitle(), ep);
                mediaList.put(ep.getTitle(), ep);
            }
        }
        return serie;
    }
```
Vi byggede disse metoder op omkring en beslutning eller en konvention vi havde vedtaget. Nemlig at vores datafiler var formateret på den måde, at første linje bestod af en header, og alle efterfølgende linjer var data. Data var adskilt af semikolon, der var et bestemt antal felter (kolonner), og de kom i en bestemt rækkefølge. Først navnet, så årstallet, så genren, osv.

Vores `FileIO` så således ud:

````java
public class FileIO
{
    public ArrayList<String> readData(String path)
    {
        ArrayList<String> dataList = new ArrayList<>();
        File file = new File(path);
        if (!file.exists())
        {
            saveData("header", new ArrayList<>(), path);
        }
        try
        {
            Scanner scan = new Scanner(file);
            scan.nextLine();//skip header

            while (scan.hasNextLine())
            {
                String line = scan.nextLine(); // Car specifics
                dataList.add(line);
            }

        } catch (FileNotFoundException e)
        {
            System.out.println("File not found");
        }
        return dataList;
    }
}
````

En ulempe ved dette er at det er svært at finde en bestemt Film. Vi er nødt til at læse hele filen ind, og kigge linje for linje ind til vi finder den ønskede Film. Hvis vi har mange film, kan det være problematisk at beholde alle film i hukommelsen, det ville være smart hvis vi kun hentede de data ind i programmet, som programmet har brug for lige nu. (Og måske de data som vi tror programmet får brug for lige om lidt.)

## En smartere måde

Databaser er designet til at opbevare en masse data på en smart måde. Og de har bl.a. muligheder for at vi kan søge i data, og kun hente de data vi beder om. Problemet med databaser er at de ikke snakker Java, de snakker deres eget sprog (SQL).

Hvor headeren i tekstfilen så ud på denne måde `Title;releaseyear;Genre;Rating;`, så kunne en tabel i en database se ud på følgende måde:

```SQL
                                         Table "public.movies"
   Column     |            Type          | Collation | Nullable |                 Default
--------------+--------------------------+-----------+----------+-----------------------------------------
 movie_id     | integer                  |           | not null | nextval('movies_movie_id_seq'::regclass)
 movie_title  | character varying(45)    |           | not null |
 movie_genre  | character varying(127)   |           | not null |
 releaseyear  | integer                  |           | not null | 
 rating       | integer                  |           | not null | 
Indexes:
    "movies_pkey" PRIMARY KEY, btree (movie_id)
    "idx_movie_title" btree (movie_title)
Referenced by:
    TABLE "film_actor" CONSTRAINT "film_actor_actor_id_fkey" FOREIGN KEY (actor_id) REFERENCES actor(actor_id) ON UPDATE CASCADE ON DELETE RESTRICT
Triggers:
    last_updated BEFORE UPDATE ON actor FOR EACH ROW EXECUTE FUNCTION last_updated()
```

Vi kan se en kolonne `movie_title`, som svarer til vores `Title` felt, osv.

### Læs fra databasen

Den gang vi læste data fra filen, havde vi en IO pakke med en `readFile` metode. Nu har vi i stedet en `DatabaseConnector` som svarer til at vi åbner filen. Så har vi en `DataMapper` som både beder databasen om at sende nogle rækker (baseret på nogle kriterier) og derefter parser data og laver objekter som vores program kan bruge.

`DatabaseConnector` vil jeg sige lidt mere om senere, men først kigger vi kort på en `MovieMapper`.

````java
import ...

public class MovieMapper
{
    private final DatabaseConnector dbConnector;

    public MovieMappper(DatabaseConector dbConnector)
    {
        this.dbConnector = dbConnector;
    }
    
    /*
            Asks the database for at list of movies with movie_title matching parameter title.
            Returns a list of movie objects.
     */
    public List<Movie> getMoviesByTitle(String title) throws DatabaseExeption
    {
        List<Movie> movies = new ArrayList<>();
        //Den rå SQL statement, dog uden vores søgeparameter
        String sql = "SELECT * FROM movies WHERE movie_title LIKE ?";
        try (
                Connection connection = dbConnector.getConection();
                var prepareStatement = connection.prepareStatement(sql)
        )
        {
            //Indsæt vores søgeparameter i SQL statement
            prepareStatement.setString(1, title);
            var resultSet = prepareStatement.executeQuery();
            while (resultSet.next())
            {
                int movieId = resultset.getInt("movie_id");
                String movieTitle = resultSet.getString("movie_title");
                String movieGenre = resultSet.getString("movie_genre");
                int releaseYear = resultSet.getInt("releaseyear");
                int rating = resultSet.getInt("rating");
                movies.add(new Movie(movieId, movieTitle, movieGenre, releaseYear, rating));
            }
        } catch (SQLException e)
        {
            throw new DatabaseException("Could not get movies from database", e);
        }
        return movies;
    }
}
````

Der sker rigtig mange ting, så lad os holde tungen lige i munden. I vores gamle `FileIO.readData(pathToFile)` havde vi en Scanner, som læste fra filen, og et `while (scan.hasNextLine())`, som gav os data en linje ad gangen. 

I vores nye `getMoviesByTitle` kommer data fra vores database ind i et `resultSet` og vi har en `while (resultSet.next())`, som giver os data en række ("linje") ad gangen.

I vores gamle `createMovieFromString` tager vi linjen, som en tekststreng, og klipper den op i de bidder vi skal bruge, og konverterer til de rigtige datatyper (`parseInt` osv.). I vores nye `getMoviesByTitle`(inde i while løkken) bruger vi database driverens indbyggede metoder (`getInt`, `getString` m.fl.) til at klippe data ud i de stykker vi har brug for, med de rigtige datatyper. Til sidst opretter vi java objektet og tilføjer det til vores liste (eller anden beholder, fx. `Map` eller `Set`).

SQL kan være et lidt tricky sprog, og databasen gør præcis hvad man beder den om. Derfor skal man passe på hvordan man skrive sine SQL statements. Især hvis man har nogle user-input, som skal bruges til at søge noget frem fra databasen. Vi skal være helt sikre på at vi ikke tillader nogle ballademagere at lave såkaldte SQL injections, og derved kommer til at køre nogle SQL kommandoer som vi ikke ønsker. Derfor bruger vi `prepareStatement` og sender vores input gennem `prepareStatement.setString()`(og `setInt()` osv.) først.

### try-catch helvedet og en hjemmelavet Exception

Den gang, sidste semester, vi lavede vores FileIO var vi ikke så skarpe i fejlhåndtering og exceptions. Men nu er vi godt i gang med at blive 100-meter mestre.

Vi har lavet vores egen DatabaseException fordi vi skal håndtere en masse metodekald som kan resultere i Exceptions. Mens vi udvikler vil vi gerne have så mange informationer med som muligt, men når vi har rettet de fleste af vores fejl, og programmet skal sendes til kunden, vil vi gerne skrue ned for mængden af rød tekst i vores fejlbeskeder. 

Derfor er det smart hvis der kun er et sted, hvor vi skal fjerne vores `e.printStackTrace()`, og nøjes med at printe en mere brugervenlig besked. Ligesom Tess forsøgte at lære os at bruge en `UI` klasse til at printe beskeder til brugeren, i stedet for at bruge `sout`.

Læg mærke til, at vi kan slå nestede `try` statements sammen