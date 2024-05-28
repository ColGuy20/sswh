# Main
The purpose of this rust program is to take in data every 10 minutes from the [Score Saber API](https://scoresaber.com/api).
## Set-up
Before starting, certain dependencies and imports are required for the program to function appropriately.
### Dependencies
The dependencies in the cargo.toml are:
```
reqwest = { version = "0.11", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
```
### Imports
The program uses 5 imports:
```
use serde::{Deserialize, Serialize}; 
use serde_json::json;
use reqwest::Error;
use tokio::time::{sleep, Duration}; 
use std::env;
```
- Serde, more specifically `serde::{Deserialize, Serialize}`, is used to serialize data, or convert for easy storage and managment. This is used when creating the structs that store the data fetched from Score Saber.
- Serde Json, or `use serde_json::json;`, is used to create JSon files. This is used to create the JSon file that is formatted for discord and sent to the webhook.
- Reqwest error, or `use reqwest::Error;`, is helpful when handling errors from HTTP requests. This is used to provide feedback when an operation, such as fetching the data or sending to discord, fails.
- Tokio time, or `use tokio::time::{sleep, Duration}; `, is used for asynchronous and time-based operations. This is used for the async functions and the sleep() function used to wait 10 minutes.
- Standard environment, or `use std::env;`, is used to interact with environmental variables. This is used to take in the WEBHOOK_URL variable that is hidden for security concerns.
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

Next, the first line of a struct, in this case `struct PlayerData` is used to name the struct.

Lastly, in this specific struct the fields are named like in `id: String`. The first part is to choose the name of the field and the second is to choose the data type.
## Functionality
The following will be brief descriptions on how the rest of the code operates.
# Fetch Player Data
This async function serves to extract data from the [Score Saber API](https://scoresaber.com/api/player/76561199396123565/full) and store it in the struct of PlayerData (See [ScoreSaber Public API](https://docs.scoresaber.com)).
```
async fn fetch_player_data() -> Result<PlayerData, Error> { //Name of function and stating return type
    let url = "https://scoresaber.com/api/player/76561199396123565/full"; //The url used to take in date (In this case it is my username)
    let response = reqwest::get(url).await?.json::<PlayerData>().await?; //Line to actually take the data in
    Ok(response) //Returns response
}
```
# Send Data to Discord
This async function starts off by taking in the webhook url from the environment and using it to state the client. Then, the functions formats the data accordingly via use of the embed discord webhook format (See [Discord Webhooks Guide](https://birdie0.github.io/discord-webhooks-guide/discord_webhook.html)). Next, it actually sends the message and checks if it was a success. Lastly, if the code functioned then it returns `OK(())`.
```
async fn send_to_discord(data: &PlayerData) -> Result<(), Error> { //Name of function and stating return type
    let webhook_url = env::var("WEBHOOK_URL").unwrap_or_else(|_| String::from("Invalid webhook URL")); //Pulls WEBHOOK_URL from environment, if unable then prints invalid
    if webhook_url == "Invalid webhook URL" {
        println!("{}", webhook_url);
    }
    
    let client = reqwest::Client::new(); //Setting the client
    
    let firstSeen_formatted = &data.firstSeen[0..10]; //Condencing the firstSeen variable to exclude time

    //Formatting the data being sent
    let payload = json!({ //The data is put in a JSon file for the discord webhook
        "embeds": [{
            "author": {
                "name": format!("{} #{}", data.name, data.rank), //Username and global rank
                "icon_url": format!("{}",data.profilePicture) //Profile Picture
            },
            "color": 5505024,
            "fields": [
                {
                    "name": "Description",
                    "value": format!("Country Rank: **#{}** ({})\nFirst Seen: {}", data.countryRank, data.country, firstSeen_formatted) //Country, Country Rank, and date first seen
                },
                {"name": "","value": ""}, //These are just for cosmetic purposes, they ensure that only two fields are in each row
                {
                    "name": "Total Score",
                    "value": format!("{}", data.scoreStats.totalScore), //Total Score
                    "inline": true //This states that this field and up to 2 other fields are in the same row
                },
                {
                    "name": "Total Ranked Score",
                    "value": format!("{}", data.scoreStats.totalRankedScore), //Total Ranked Score
                    "inline": true             
                },
                {"name": "","value": ""},
                {
                    "name": "Average Ranked Accuracy",
                    "value": format!("%{}", data.scoreStats.averageRankedAccuracy), //Average Ranked Accuracy
                    "inline": true
                },
                {
                    "name": "Performance Point (PP)",
                    "value": format!("{}", data.pp), //Performance Points
                    "inline": true
                },
                {"name": "","value": ""},
                {
                    "name": "Ranked Play Count",
                    "value": format!("{}", data.scoreStats.rankedPlayCount), //Ranked Play Count
                    "inline": true
                },
                {
                    "name": "Total Play Count",
                    "value": format!("{}", data.scoreStats.totalPlayCount), //Total Play Count
                    "inline": true             
                }
            ]
        }]
    });
    
    //Sends to discord
    let response = client.post(webhook_url)
        .json(&payload)
        .send()
        .await?;
    
    //Checks if it worked properly
    if response.status().is_success() {
        println!("Message sent successfully to Discord"); //Prints if successful
    } else {
        println!("Failed to send message to Discord: {}", response.status()); //Prints if failure
    }
    
    Ok(()) //Returns OK
}
```
# Main Function
This is the main method of the program. Again, this is an async function given the main property by tokio (`#[tokio::main]`). This loops every ten minutes by using `sleep(Duration::from_secs(600)).await;` imported from tokio. Lastly, if checks twice for errors and responds accordingly.
```
#[tokio::main]
async fn main() {
    loop { //Loops every 10 minutes
        match fetch_player_data().await { //Checks if fetching data goes successfully
            Ok(data) => {
                if let Err(e) = send_to_discord(&data).await {
                    println!("Error sending to Discord: {}", e); //Prints if sending to discord results in failure
                }
            },
            Err(e) => println!("Error: {}", e), //Prints if fetching data results in failure
        }
        
        sleep(Duration::from_secs(600)).await; //Waits 10 minutes before looping
    }
}
```
## Credits
- Website used for score saber webhook generation and guide: https://docs.scoresaber.com
- Website used for formatting JSon to send to the discord webhook: https://birdie0.github.io/discord-webhooks-guide/discord_webhook.html
- Website used for converting RGB and HEX into decimal: https://www.mathsisfun.com/hexadecimal-decimal-colors.html
- ScoreSaber official website: https://scoresaber.com
- ScoreSaber discord: https://discord.com/invite/scoresaber
- Official GitHub Guide for github syntax: https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax
- Big Thanks to coinchimp for all the support: https://github.com/coinchimp/