
# fanSALE Diagnostic

A brief description look at why Fansale's sale of Radiohead tickets aren't 'working'. There is going to be a lot of technical talk, however, I will provide an 'ELI5' at the end. But for those who enjoy the the technical talk, enjoy the read!

#  Introduction

For clarification, I chose a random event on fanSALE to make this comparison and randomly chose Apache 207 (the same diagnostic is the same for any other events) could still be purchased on fanSALE while Radiohead’s page showed tickets as available but failed at checkout. So, I opened Chrome’s Developer Tools and began investigating the network calls, console logs, and embedded JavaScript on both pages.

# Step 1 — Starting with Apache 207

I began with the Apache 207 series page. As soon as I loaded it, I saw `“Angebote ab …”` (offers from €X) beneath each city/date row — clear signs of active listings.

In Developer Tools → Network, clicking on one of those rows triggered multiple JSON requests to Eventim’s fanSALE API, fetching offer data and sometimes the seatmap image overlays.

Inspecting the DOM confirmed this. The page was filled with structured metadata like:
```html
<meta itemprop="price" content="120.00" />
<meta itemprop="availability" content="https://schema.org/InStock" />
```
With this in mind, I knew the Apache 207 listings were genuinely live.

When I expanded the network calls (live data flowing from my network to the server's network), I found endpoints like: `https://public-api.eventim.com/seatmap/api/public/seatmap/fansale-1001-3732416/0?seats=false&blockTexts=false&rowLabels=false&hulls=false&images=true&a_affiliate=FAN&a_systemId=1`

The seatmap rendered properly on the listings for Apache 207, confirming that this event’s seatmap configuration was active.

# Step 2 - Moving to Radiohead

Then I opened the Radiohead event page for comparison. It immediately behaved differently.

Instead of showing offers, I saw the message: 
`“Es wurden leider keine passenden Angebote gefunden.”`

A modal appeared prompting me to click “Angebote laden” — fanSALE’s bot‑protection gate. Even after passing it, no offers loaded. Looking at the JavaScript console, I noticed this snippet during initialization:


```javascript
FS.eventdata.eventdetailspage.eventDetailsPageUi = new FS.eventdata.eventdetailspage.EventDetailsPageUI(
  "20529009",
  "/tickets/all/radiohead/520/20529009",
  {
    noSeatmap: true,
    seatmapWithOnlyGA: false,
    seatmapWithSeats: false,
    singleLayer: false,
    multiLayer: false
  }
);
```

Importantly, that `noSeatmap: true`, caught my attention. It effectively disables the seatmap rendering (what makes the arena map show). I confirmed this by searching for more API calls to the seatmap endpoint in the Network...There were zero.

# Step 3 - Checking the seatmap configuration block

What was curious to me, the page still contained this block:
```html
<script>
FS.svgSeatmap.additionalSvgSeatmapConfig = {
baseUrl: "https://public-api.eventim.com/seatmap/api/public",
affiliateCode: "FAN",
connectorId: "1001",
connectorType: "fansale",
seatmapEventId: "4064107",
seatmapSystemId: "1"
};
</script>
```

At the start, I was just assuming the map was slow to load. But since `noSeatmap` was set to `True`, the seatmap's initialization code never executed, therefore the API call never fired. I tried manually refreshing the endpoint returned the same thing: `{"errorCode":"400-SMS-061","detail":"TDL connection error"}`

That confirmed to me that the backend wasn’t serving a valid seatmap for this event anymore.

# Step 4 - Testing the purchase links

Next, I tried to force checkout via a deep link: `https://www.fansale.de/tickets/all/radiohead/520/20529009?offerId=10141456&ptc=1`

The result? The error message everybody is seeing: “Das Angebot kann gerade nicht gekauft werden. Bitte habe ein wenig Geduld und versuche es später erneut.”

Looking back at the JavaScript, I found this preloaded string:
```js
FS.base.messages.prepareByList([
["eventDetails.detailCSection.automatedReprintErrorMessage",
"Das Angebot kann gerade nicht gekauft werden. Bitte habe ein wenig Geduld und versuche es später erneut."]
]);
```
The message was not random. It gets triggered when an offer fails validation or when automated reprint (Digital Reissue) fails internally (on their side, not yours)

Given that the Radiohead tickets sold out quite a bit ago, I concluded that the `offerId` I tried was "stale" or withdrawn, and the event was flagged with an automated reprint delivery that no longer works.

# Step 5 - Testing the purchase links

Here's what can be observed side-by-side:

# Apache 207:
`event_seatmap_available` - true

`noSeatmap` in JS - `false`

Seatmap API Call - ✅ Yes


Offers available - ✅ Yes (active listings)


Bot gate - ✅ Yes (active listings)


Checkout works - ✅ Valid offer → basket

# Radiohead:
`event_seatmap_available` - false

`noSeatmap` in JS - `true`

Seatmap API Call - ❌ None


Offers available - ❌ None (empty state)


Bot gate - Always shown (no offers)


Checkout works - ❌ Fails with automated message

# Step 6 - Conclusions:
The Radiohead event is archived: its seatmap API fails (`TDL connection error`), `noSeatmap` is set to `true`, and no current offers exist. They basically haven't removed the purchase page, likely they are keeping it up in case they decide to add more tickets later on.

The Apache 207 page is live: seatmap initialized, active listings fetched, and checkout flow functions end‑to‑end.

The front‑end error message seen on Radiohead is not caused by bot‑protection,...It is a fallback for expired or blocked listings, or as I said, they haven't taken down the listings in case they want to add more later on.

# My Final Insight
The fanSALE frontend exposes its backend state very transparently. By reading the initialization flags and watching API calls in Chrome's DevTools, it’s clear that Radiohead’s page is disabled at source, not a bug, but a reflection that no valid tickets or seatmap exist for that event anymore. Apache 207, on the other hand, is still actively syncing inventory with Eventim’s seatmap and checkout APIs.

# ELI5
Apache 207’s page still connects to Eventim’s servers. It loads real offers and a working seatmap, so you can buy tickets.
Radiohead’s page is shut off. The seatmap is disabled, no offers load, and checkout links just trigger the “can’t be purchased” message because those tickets no longer exist.

Thanks for reading!

-omgitsmint
