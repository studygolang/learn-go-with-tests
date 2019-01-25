# Websockets (WIP)

In this chapter we'll learn how to use websockets to improve our application. 

## Project recap

We have two applications in our poker codebase

- *Command line app*. Prompts the user to enter the number of players in a game. From then on informs the players of what the "blind bet" value is, which increases over time. At any point a user can enter `"{Playername} wins"` to finish the game and record the victor in a store.
- *Web app*. Allows users to record winners of games and displays a league table. Shares the same store as the command line app. 

## Next steps

The product owner is thrilled with the command line application but would prefer it if we could bring that functionality to the browser. She imagines a web page with a text box that allows the user to enter the number of players and when they submit the form the page displays the blind value and automatically updates it when appropriate. Like the command line application the user can declare the winner and it'll get saved in the database.

On the face of it, it sounds quite simple but as always we must emphasise taking an _iterative_ approach to writing software.

First of all we will need to serve HTML. So far all of our HTTP endpoints have returned either plaintext or JSON. We _could_ use the same techniques we know (as they're all ultimately strings) but we can also use the [/html/template](https://golang.org/pkg/html/template/) for a cleaner solution.

We also need to be able to asynchronously send messages to the user saying `The blind is now *y*` without having to refresh the browser. We can use [websockets](https://en.wikipedia.org/wiki/WebSocket) to facilitate this. 

> WebSocket is a computer communications protocol, providing full-duplex communication channels over a single TCP connection

Given we are taking on a number of techniques it's even more important we do the smallest amount of useful work possible first and then iterate. 

For that reason the first thing we'll do is create a web page with a form for the user to record a winner. Rather than using a plain form, we will use websockets to send that data to our server for it to record. 

After that we'll work on the blind alerts by which point we will have a bit of infrastructure code set up. 

### What about tests for the JavaScript ?

There will be some JavaScript written to do this but I wont go in to writing tests. 

It is of course possible but for the sake of brevity I wont be including any explanations for it. 

Sorry folks. Lobby O'Reilly to pay me to make a "Learn JavaScript with tests".

## Write the test first

First thing we need to do is serve up some HTML to the user when they hit `/game`. 

Here's a reminder of the pertinent code in our web server

```go
// PlayerServer is a HTTP interface for player information
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

// NewPlayerServer creates a PlayerServer with routing configured
func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}
```

The _easiest_ thing we can do for now is check when we `GET /game` that we get a `200`.

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request, _ := http.NewRequest(http.MethodGet, "/game", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

## Try to run the test
```
--- FAIL: TestGame (0.00s)
=== RUN   TestGame/GET_/game_returns_200
    --- FAIL: TestGame/GET_/game_returns_200 (0.00s)
    	server_test.go:109: did not get correct status, got 404, want 200
```

## Write enough code to make it pass

Our server has a router setup so it's relatively easy to fix. 

To our router add

```go
router.Handle("/game", http.HandlerFunc(p.game))
```

And then write the `game` method

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}
```

## Refactor

The server code is already fine due to us slotting in more code into the existing well-factored code very easily. 

We can tidy up the test a little by adding a helper (write this yourself) to make the request to `/game`

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request :=  newGameRequest()
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response, http.StatusOK)
	})
}
```

You'll also notice I changed `assertStatus` to accept `response` rather than `response.Code` as I feel it reads better.

Now we need to make the endpoint return some HTML, here it is

```html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Lets play poker</title>
</head>
<body>
<section id="game">
    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>
</section>
</body>
<script type="application/javascript">

    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    if (window['WebSocket']) {
        const conn = new WebSocket('ws://' + document.location.host + '/ws')

        submitWinnerButton.onclick = event => {
            conn.send(winnerInput.value)
        }
    }
</script>
</html>
```

We have a very simple web page with a text input for the user to enter into and a button they can click to declare the winner. 

`WebSocket` is built into most modern browsers so we don't need to worry about bringing in any libraries. The web page wont work for older browsers, but we're ok with that for this scenario.

Our code uses the `WebSocket` API to open a connection to our server. We then attach an `onclick` handler to our button to send the contents of the input button to the server through the connection.

### How do we test we return the correct markup?

There are a few ways. As has been emphasised throughout the book, it is important that the tests you write have sufficient value to justify the cost.

1. Write a browser based test, using something like Selenium. These tests are the most "realistic" of all approaches because they start an actual web browser of some kind and simulates a user interacting with it. These tests can give you a lot of confidence your system works but are more difficult to write than unit tests and much slower to run. For the purposes of our product this is overkill.
2. Do an exact string match. This _can_ be ok but these kind of tests end up being very brittle. The moment someone changes the markup you will have a test failing when in practice nothing has _actually broken_.
3. Check we call the correct template. We will be using a templating library from the standard lib to serve the HTML (discussed shortly) and we could inject in the _thing_ to generate the HTML and spy on its call to check we're doing it right. This would have an impact on our code's design but doesn't actually test a great deal; other than we're calling it with the correct template file. Given we will only have the one template in our project the chance of failure here seems low. 

So in the book "Learn Go with tests" for the first time, we're not going to write a test. 

Put the markup in a file called `game.html`

Next change the endpoint we just wrote to the following

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	tmpl, err := template.ParseFiles("game.html")

	if err != nil {
		http.Error(w, fmt.Sprintf("problem loading template %s", err.Error()), http.StatusInternalServerError)
		return
	}
	
	tmpl.Execute(w, nil)
}
```

[`html/template`](https://golang.org/pkg/html/template/) is a Go package for creating HTML. In our case we call `template.ParseFiles`, giving the path of our html file. Assuming there is no error you can then `Execute` the template, which writes it to an `io.Writer`. In our case we want it to `Write` to the internet, so we give it our `http.ResponseWriter`. 

As we have not written a test, it would be prudent to manually test our web server just to make sure things are working as we'd hope. Go to `cmd/webserver` and run the `main.go` file. Visit `http://localhost:5000/game`. 

You _should_ have got an error about not being able to find the template. You can either change the path to be relative to your folder, or you can have a copy of the `game.html` in the `cmd/webserver` directory. I chose to create a symlink (`ln -s ../../game.html game.html`) to the file inside the root of the project so if I make changes they are reflected when running the server. 

If you make this change and run again you should see our UI. 

Now we need to test that when we get a string over a web socket connection to our server that we assume it is a winner of a game.

## Write the test first

For the first time we are going to use an external library so that we can work with WebSockets. 

Run `go get github.com/gorilla/websocket`

This will fetch the code for the excellent [Gorilla WebSocket](https://github.com/gorilla/websocket) library.

```go
t.Run("when we get a message over a websocket it is a winner of a game", func(t *testing.T) {
    store := &StubPlayerStore{}
    winner := "Ruth"
    server := httptest.NewServer(NewPlayerServer(store))
    defer server.Close()

    wsURL := "ws" + strings.TrimPrefix(server.URL, "http") + "/ws"

    ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
    if err != nil {
        t.Fatalf("could not open a ws connection on %s %v", wsURL, err)
    }
    defer ws.Close()

    if err := ws.WriteMessage(websocket.TextMessage, []byte(winner)); err != nil {
        t.Fatalf("could not send message over ws connection %v", err)
    }

    AssertPlayerWin(t, store, winner)
})
```

Make sure that you have an import for the `websocket` library. My IDE automatically did it for me, so should yours.

To test what happens from the browser we have to open up our own WebSocket connection and write to it. 

Our previous tests around our server just called methods on our server but now we need to have a persistent connection to our server. To do that we use `httptest.NewServer` which takes a `http.Handler` and will spin it up and listen for connections. 

Using `websocket.DefaultDialer.Dial` we try to dial in to our server and then we'll try and send a message with our `winner`

Finally we assert on the player store to check the winner was recorded.

## Try to run the test
```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:124: could not open a ws connection on ws://127.0.0.1:55838/ws websocket: bad handshake
```

We have not changed our server yet to accept WebSocket connections on `/ws`.

## Write enough code to make it pass

Add another listing to our router

```go
router.Handle("/ws", http.HandlerFunc(p.webSocket))
```

Then add our new `webSocket` handler

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	upgrader.Upgrade(w, r, nil)
}
```

To accept a WebSocket connection we `Upgrade` it. If you now re-run the test you should move on to the next error.

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:132: got 0 calls to RecordWin want 1
```

Now that we have a connection opened, we'll want to listen for a message and then record it as the winner.

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	conn, _ := upgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

(Yes, we're ignoring a lot of errors right now!)

`conn.ReadMessage()` blocks on waiting for a message on the connection. Once we get one we use it to `RecordWin`. This would finally close the WebSocket connection. 

If you try and run the test, it's still failing. 

The issue is timing. There is a delay between our WebSocket connection reading the message and recording the win and our test finishes before it happens. You can test this by putting a short `time.Sleep` before the final assertion. Let's go with that for now.

```go
time.Sleep(10 * time.Millisecond)
AssertPlayerWin(t, store, winner)
``` 

## Refactor

We committed many sins to make this test work both in the server code and the test code. 

Let's start with the server code.

We can move the `upgrader` to a private value inside our package because we don't need to redeclare it on every WebSocket connection request

```go
var wsUpgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

Finally in our test code we can create a helper to tidy up sending messages

```go
func writeWSMessage(t *testing.T, conn *websocket.Conn, message string) {
	if err := conn.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}
}
```

Now the tests are passing try running the server and declare some winners in `/game`. You should see them recorded in `/league`. Remember that every time we get a winner we _close the connection_, you will need to refresh the page to open the connection again.


-------

note to self: The key is to refactor `game` so that start takes a destination as to where to write the blind things


final markup..

```html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Lets play poker</title>
</head>
<body>
<section id="game">
    <div id="game-start">
        <label for="player-count">Number of players</label>
        <input type="number" id="player-count"/>
        <button id="start-game">Start</button>
    </div>

    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>

    <div id="blind-value"/>
</section>

<section id="game-end">
    <h1>Another great game of poker everyone!</h1>
    <p><a href="/league">Go check the league table</a></p>
</section>

</body>
<script type="application/javascript">
    const startGame = document.getElementById('game-start')

    const declareWinner = document.getElementById('declare-winner')
    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    const blindContainer = document.getElementById('blind-value')

    const gameContainer = document.getElementById('game')
    const gameEndContainer = document.getElementById('game-end')

    declareWinner.hidden = true
    gameEndContainer.hidden = true

    document.getElementById('start-game').addEventListener('click', event => {
        startGame.hidden = true
        declareWinner.hidden = false

        const numberOfPlayers = document.getElementById('player-count').value

        if (window['WebSocket']) {
            const conn = new WebSocket('ws://' + document.location.host + '/ws')

            submitWinnerButton.onclick = event => {
                conn.send(winnerInput.value)
                gameEndContainer.hidden = false
                gameContainer.hidden = true
            }

            conn.onclose = evt => {
                blindContainer.innerText = 'Connection closed'
            }

            conn.onmessage = evt => {
                blindContainer.innerText = evt.data
            }

            conn.onopen = function () {
                conn.send(numberOfPlayers)
            }
        }
    })
</script>
</html>
```
