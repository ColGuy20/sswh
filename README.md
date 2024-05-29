# Main
The purpose of this rust program is to take in data every 10 minutes from the [Score Saber API](https://scoresaber.com/api).
- Last updated: 2024-05-29
- Last commit version: (1.2.2)
## Set-up
Before starting, certain dependencies and imports are required for the program to function appropriately.
### Dependencies
The dependencies in the cargo.toml are:
```
reqwest = { version = "0.11", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
rusqlite = { version = "0.28.0", features = ["bundled"] }
```
### Imports
The program uses 5 imports:
```
use serde::{Deserialize, Serialize}; 
use serde_json::json;
use reqwest::Error;
use tokio::time::{sleep, Duration}; 
use std::env;
use rusqlite::{params, Connection, Result};
```
- Serde, more specifically `serde::{Deserialize, Serialize}`, is used to serialize data, or convert for easy storage and managment. This is used when creating the structs that store the data fetched from Score Saber.
- Serde Json, or `serde_json::json`, is used to create JSon files. This is used to create the JSon file that is formatted for discord and sent to the webhook.
- Reqwest error, or `reqwest::Error`, is helpful when handling errors from HTTP requests. This is used to provide feedback when an operation, such as fetching the data or sending to discord, fails.
- Tokio time, or `tokio::time::{sleep, Duration} `, is used for asynchronous and time-based operations. This is used for the async functions and the sleep() function used to wait 10 minutes.
- Standard environment, or `std::env`, is used to interact with environmental variables. This is used to take in the WEBHOOK_URL variable that is hidden for security concerns.
-Rust Sqlite, or `rusqlite::{params, Connection, Result}`, is used to integrate sqlite (extension for databases) into rust. This is used to create, add data, and fetch data from the player_data database.
## Requirements
> [!WARNING]
> Key information users need to run the program. This part of the set-up needed its own section due to them varying depending on the user, system, and use. Make sure you follow these steps to ensure the program functions accordingly.
### Running the program
**When your program is ready to run type in: `.\start.cmd` into the terminal. Make sure that you are inside the `\sswh` directory.** This calls the start.cmd which is an essential part of this program because it will **NOT** function without it.
The start command should look like this:
```
cargo build -r
set LIB=PATH_TO_SQLITE3_LIB\sqlite3;%LIB%
set WEBHOOK_URL=YOUR_URL
start PATH_TO_SSWH\sswh.exe
```
The first line `cargo build -r` simply builds the cargo. The next line will be skipped for now and will be referenced in the Database category next up. As for the last two, the first one sets the `WEBHOOK_URL` to your discord webhook's url, so you will have to replace `YOUR_URL`. As for the last one, this is the line that actually runs the program. Make sure to replace `PATH_TO_SSWH` with your own path to sswh. If you are sharing this, then it is recommended to add the start.cmd to your `.gitignore` (An example would be `start*`). 
### Setting up the Database (SQLite)
To set up the data base for your program follow along these steps:
1. Go to the official [SQLite Download Page](https://www.sqlite.org/download.html). Under "Precompiled Binaries for Windows" download **"sqlite-dll-win-x64-3460000.zip"** (64-bit) or the appropriate version.
2. Extract the ZIP file, this should give you the **sqlite3.dll** and the **sqlite3.def** files.
3. Move the files into a know location. A good place would be a new folder called **sqlite3** found in `C:\sqlite3`.
4. To generate the **.lib** file, launch **Developer Command Prompt for Visual Studio** and navigate to the directory where your files are located. Then run the following command: `lib /def:sqlite3.def /out:sqlite3.lib /machine:x64`. If done properly, this will generate the **sqlite3.lib** file.
5. Back in your `start.cmd` file, make sure to add the line: `set LIB=PATH_TO_SQLITE3_LIB\sqlite3;%LIB%`. This adds the directory to your library path.
6. Make sure you added the dependency to your `cargo.toml` and you imported the rusqlite correctly into the program. This is already done but it is always better to double check.
This just adds the directory to the library path.
## Structs
Structs are a way to layout and group together related pieces of data. They provide a way of grouping distinct data types into one organized data stucture. In this case, the structs were used to store the data taken in from the Score Saber API.

There are two structs in this program. The second is the main struct and used to store **all** the data. The first is a struct within a struct used to organize the data inside of ScoreStats to make the fetching more efficient.
```
#[derive(Debug, Serialize, Deserialize)] //This stores data under ScoreStats
struct ScoreStats {
    totalScore: i64,
    totalRankedScore: i64,
    averageRankedAccuracy: f64,
    totalPlayCount: i64,
    rankedPlayCount: i64,
    replaysWatched: i64,
}
#[derive(Debug, Serialize, Deserialize)] //This is where all the data is stored (PlayerData)
struct PlayerData {
    id: String,
    name: String,
    profilePicture: String,
    country: String,
    pp: f64,
    rank: i64,
    countryRank: i64,
    histories: String,
    banned: bool,
    inactive: bool,
    scoreStats: ScoreStats,
    firstSeen: String,
}
```
The `#[derive(Debug, Serialize, Deserialize)]` is used to provide the three properties to the struct. The most important are serialize and deserialize because they allow the alteration of the data into a JSon file, formatted for the webhook.

Next, the first line of a struct, in this case `struct PlayerData`, is used to name the struct.

Lastly, in this specific struct the fields are named like in `id: String`. The first part is to choose the name of the field and the second is to choose the data type.
## Functionality
The following will be brief descriptions on how the rest of the code operates.
# Fetch Player Data from ScoreSaber
This async function serves to extract data from the Score Saber API and store it in the struct of PlayerData (See [ScoreSaber Public API](https://docs.scoresaber.com)). It uses the url to take in the data as a json (via `reqwest::get` and `.json`) and returns it structured in the `PlayerData` struct.
```
async fn fetch_player_data() -> Result<PlayerData, Error> { //Name of function and stating return type
    let url = "https://scoresaber.com/api/player/76561199396123565/full"; //The url used to take in date (In this case it is my username)
    let response = reqwest::get(url).await?.json::<PlayerData>().await?; //Line to actually take the data in
    Ok(response) //Returns response
}
```
# Fetch Player Data from DB
This function serves to fetch the data inside of the database and store it in the same struct as the previous function (`fetch_player_data`). It takes in the connection to the database and returns the data formatted in the `PlayerData` struct. First, it begins by preparing the query (a request for data) by stating what data it is taking in, in this case it is taking in all the data in `player_data`. Then, it actually executes the query and tells where each row of data is stored/found. Then it makes a new vector (`player_data_vector`) and adds the data to it by using a for loop. Lastly, it returns `Ok(player_data_vec)`, this states that everything went well and also includes the data fetched.

First line of fetch_player_data_from_db: `fn fetch_player_data_from_db(conn: &Connection) -> Result<Vec<PlayerData>>`
# Insert Player Data
This function is made up of two parts but ultimately serves the purpose of storing the fetched data into a database. The first part creates a table if none exist yet. This table is inside the connection and named, you guessed it, `player_data`. The second part either inserts or replaces the data inside the databased with the new, updated data fetched from the API. It finishes off by returning `OK(())` to show that everything went well.

First line of insert_player_data: `fn insert_player_data(conn: &Connection, data: &PlayerData) -> Result<()> {`
# Send Data to Discord
This async function's purpose is to send the data to the discord server (via webhook). It starts off by taking in the webhook url from the environment and using it to state the client. Then, the function formats the data accordingly via use of the embed discord webhook format (See [Discord Webhooks Guide](https://birdie0.github.io/discord-webhooks-guide/discord_webhook.html)). Next, it actually sends the message and checks if it was a success. Lastly, if the code functioned then it returns `OK(())`.

First line of send_to_discord: `async fn send_to_discord(data: &PlayerData, success_count: &mut i64) -> Result<(), Error> {`
# Main Function
This is the main method of the program. Again, this is an async function given the main property by tokio (`#[tokio::main]`). It begins by creating two things: the `success_count` and connection for the database (`let conn = Connection::open("player_data.db").expect("Failed to open database");`). It then enters a loop that repeats every ten minutes by using `sleep(Duration::from_secs(600)).await;` imported from tokio. In the loop, it calls and checks the fetch_player_data, insert_player_data, and send_to_discord functions. It responds accordingly depending on if they functioned or had failed before looping.
## Additional Functions
This category are function that aren't necessarily needed (cosmetic, formatting, extra, etc.).
# Add Comma
This function serves one purpose, to add commas to the number taken in. It takes in an i64 and returns a String. It begins by making an integer for the count (`count`) and the answer (`result`). Then, as long as there is more to the number, it begin to loop (`while num > 0`). If the count is divisible by three, then a comma is added. Next, no matter the digit, it is added to the `result`. After being the digit is added to the answer, the next number is divided by 10 to check the next digit and the count is incremented. The function finishes by returning the answer (`result`).

First line of add_comma: `fn add_commas(mut num: i64) -> String`
## Brief Log
V1.2 (DB Integrated)
- <1.2.2> Added `fetch_player_data_from_db` for fetching data from DB > [See `fn fetch_player_data_from_db(conn: &Connection) -> Result<Vec<PlayerData>> {`]
- <1.2.1> Added `add_comma` function for formatting > [See `fn add_commas(mut num: i64) -> String {`]
    - Made variables for ID used in ScoreSaber API URL and cooldown for sleep function
- <1.2> Added `insert_player_data` function to store fetched data in the database (`player_data.db`) > [See `fn insert_player_data(conn: &Connection, data: &PlayerData) -> Result<()> {`]
V1.1 (Pre-DB)
- <1.1> Added `success_count` to log amount of loops > [See `let mut success_count: i64 = 0;`]
## Credits
- Website used for score saber webhook generation and guide: https://docs.scoresaber.com
- Website used for formatting JSon to send to the discord webhook: https://birdie0.github.io/discord-webhooks-guide/discord_webhook.html
- Website used for converting RGB and HEX into decimal: https://www.mathsisfun.com/hexadecimal-decimal-colors.html
- ScoreSaber official website: https://scoresaber.com
- ScoreSaber discord: https://discord.com/invite/scoresaber
- Official GitHub Guide for github syntax: https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax
- Big Thanks to @coinchimp for all the support: https://github.com/coinchimp/