# Draft Lottery Night

A web app that randomizes your fantasy football draft order with a lottery-machine show.

- Every team gets a ball in an air-blown drum, labeled with the team's initials.
- Picks are drawn from last to first. The final two face off, and the next ball out gets the No. 1 pick, with confetti.
- The order comes from a Fisher–Yates shuffle that draws from `crypto.getRandomValues`. The animation only reveals it.
- Each draw gets a short Draw ID and a timestamp, so your league can tell which result is the official one.
- The board shows each team's Round 2 and Round 3 picks for snake or linear formats.
- Use **Copy order** to paste the result into your league chat.
- Choose Dramatic or Quick reveal speed, turn sound on or off, and skip to the results at any time.
- Your team list and your last result are saved in your browser, so a refresh doesn't re-roll the draw.

## Run it

It's one static file with no build step. Open `index.html` in a browser, or host it anywhere static (for example, GitHub Pages: Settings → Pages → deploy from this branch).
