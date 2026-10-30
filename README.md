# Osysharp.Notifications

Telling a person that something needs them. A bell beside their name with a live count and a list behind it — read
and unread, All / Unread, "Mark all read", each row opening where it happened — and, for what they did not see in the
app, a mail: a daily digest, or a mention at once. Each person chooses; each person reads it in their own language.

**Use case.** A support team where a ticket is offered to Rahul while he is on another screen; a colleague writes
`@Rahul Kapoor can you check the refund?` in a note; the customer writes again on a ticket he follows; a promise on his
ticket is at risk. Each reaches his bell at once, and what he has not read by morning is in one mail. The same kit
tells an approver that a purchase waits for them, or a privacy officer that a request is due — it never knows what a
ticket is.

```osy
use Osysharp.Notifications@1;

policy ReceivesNotifications => IsAgent;                        // who has a bell
partial entity Notification { security { allow create when IsAgent; } }   // who may tell from their own request
app.Notifications = new NotificationsSetup { AppName = "Harbour" };

[Event("Customer wrote again")]                                 // what happened — your own event, or any kit's
public sealed class CustomerWrote {
  public Ticket Ticket;
  public override string Sentence() { return $"{Ticket.Customer.DisplayName} wrote again."; }   // said when READ
}

[Title($"#{Number} · {Title}")] entity Ticket { … }                       // what a ticket is called
[Route("/w/{view}/t/{number}")] [Opens(Ticket, view = "mine")] component TicketPage(…) { … }   // where it opens

Notify(rahul, new CustomerWrote { Ticket = ticket });           // about the record the event is about
[On] void TellReviewers(ArticleSentForReview e) { NotifyAll(reviewers, e); }   // a kit's event, told as it is
Mention(rahul, ticket, by: Session.CurrentUser);                // a mention is the kit's own
NotificationBell();                                             // in the shell's actions
```

## What it does

| | |
|---|---|
| **Durable delivery** | a notification is written in the caller's commit and delivered after it by a workflow (retried on a throw) — nobody hears about something rolled back |
| **Only what they may read** | the record's own security, as the recipient (`Records.MayRead`) — asked at delivery; the bell reads each record as its reader and the digest asks once per person (`Records.Readable`), so a record they lost access to leaves the bell and the mail |
| **About any record** | a notification's `About` is an `Entity`: what it is called and where it opens are the record's own `[Title]` and the page that `[Opens]` it — the kit asks nothing of the app about them — and it goes with its record (`[OnDelete(Remove)]`) |
| **One unread per thing** | a second notification with the same key joins the unread one ("2 times"), held by the database |
| **The bell** | `NotificationBell()` — count in its accessible name, a panel wide, a sheet on a phone, empty and caught-up states designed |
| **Mail** | per person, on their own settings page: mentions at once (after a short wait, only if still unread) and the rest daily; daily only; or nothing — always in their language |
| **At once, on a phone** | Web Push to every browser a person allowed, as it reaches their bell — everything, mentions only, or nothing, their own choice; signed with the app's VAPID key, carrying nothing (the browser fetches the lines as its person); tried again while a push service is down, and every try asks again whether they may read it |
| **Channels of your own** | `INotificationChannel` — a chat, a pager — named in `NotificationsSetup.Channels` beside Web Push, each send its own row and retried on its own |
| **The event, not its sentence** | a notification keeps the event itself and says it with the event's own `Sentence()` when it is read — in the reader's language, with the event's records read as they may read them; an event that says nothing of its own says its label |
| **Retention** | read ones after 30 days, unread after 180, never-shown after a day — an hourly sweep |
| **Privacy** | the event it keeps is `Personal.Content`; with `Osysharp.Privacy`, Osysharp.Privacy.Notifications — which arrives by itself — counts, copies, restricts and erases a person's notifications and preferences |

## Web Push

```osy
use Osysharp.Notifications@1 { egress "fcm.googleapis.com"; egress "updates.push.services.mozilla.com";
                               egress "*.push.apple.com"; }                // the push services browsers name
app.Secrets = [ new Secret("VapidKey") ];                                   // the VAPID private key, PEM
app.Notifications = new NotificationsSetup { AppName = "Harbour", Channels = [
  new WebPushChannel { PublicKey = "BKyBWg3t…", PrivateKey = Secret.VapidKey, Subject = "mailto:ops@harbour.example" } ] };
```

The page offers "notify me on this device" when `PushPublicKey()` is not empty, subscribes the browser with it and hands
the subscription to `SubscribeToPush(endpoint, p256dh, auth)`; the app's service worker, woken by a push, shows
`UnreadNotificationLines()` — asked as its person, so what appears is read under their own rules. The push itself says
nothing about any record. The wire format, the browser's half and making a key pair: `osy docs ui-notifications-web-push`.

## Screens

| | |
|---|---|
| The bell open, wide | ![](docs/notifications-bell-wide-light.png) |
| On a phone | ![](docs/notifications-bell-phone-light.png) |
| A mention, offered as you type | ![](docs/notifications-mention-menu-wide-light.png) |
| The daily digest | ![](docs/notifications-digest-mail.png) |

## Its tests

The kit's test app is a small studio's shared drafts, so the kit is shown working outside a helpdesk; the
[Harbour helpdesk](https://osyrin.com/templates/helpdesk/) uses it end to end. Who may read a notification is
`osy docs stdlib-records`.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Notifications;
using NotificationsKitTests;

[Test(Studio)]
void a_request_for_a_read_reaches_the_bell_once_it_is_delivered() {
  runas (Ada) { AskForARead(DraftNamed("Spring issue"), PersonAt("bo@studio.test")); }
  // Written, not yet delivered: nothing in Bo's bell.
  runas (Bo) { Assert.Equal(0, UnreadNotifications()); }
  Background.Settle();
  runas (Bo) {
    Assert.Equal(1, UnreadNotifications());
    var line = MyNotificationLines().Single();
    Assert.Equal("Ada Lindqvist asked you to read a draft.", line.Sentence);
    Assert.Equal("“Spring issue”", line.Title);
    Assert.Equal("Ada Lindqvist", line.By);
    Assert.True(line.Link.StartsWith("/drafts/"));
  }
}
```

From [`tests/delivery.test.osy`](tests/delivery.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Notifications;
using Osysharp.TimeZone;
using NotificationsKitTests;

[Test(Studio)]
void the_digest_goes_out_in_the_teams_morning_on_the_apps_clock() {
  // The studio works in Tokyo (UTC+9): 06:00 there is 21:00 UTC the evening before.
  runas (Ada) {
    new ClockSettings { TimeZone = TimeZoneInfo.FindSystemTimeZoneById("Asia/Tokyo") };
    new Draft { Title = "Summer issue", Writer = Person.Single(p => p.Email == "ada@studio.test") };
    UnitOfWork.Commit();
  }
  TestClock.Set(new DateTime(2030, 3, 3, 20, 0, 0));          // 05:00 on 4 March in Tokyo
  runas (Ada) { AskForARead(DraftNamed("Spring issue"), PersonAt("bo@studio.test")); }
  Background.Settle();
  Assert.Equal(0, MailTo("bo@studio.test").Count);

  TestClock.Set(new DateTime(2030, 3, 3, 21, 30, 0));         // 06:30 in Tokyo — the morning, though UTC is still the 3rd
  Background.Settle();
  Assert.Equal(1, MailTo("bo@studio.test").Count);

  // 06:30 UTC on the 4th is 15:30 the same day in Tokyo: one digest a day there, so nothing more.
  runas (Ada) { AskForARead(DraftNamed("Summer issue"), PersonAt("bo@studio.test")); }
  TestClock.Set(new DateTime(2030, 3, 4, 6, 30, 0));
  Background.Settle();
  Assert.Equal(1, MailTo("bo@studio.test").Count);

  // The next morning in Tokyo, the new one goes.
  TestClock.Set(new DateTime(2030, 3, 4, 21, 30, 0));
  Background.Settle();
  Assert.Equal(2, MailTo("bo@studio.test").Count);
}
```

From [`tests/mail.test.osy`](tests/mail.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Http;
using Osysharp.Mail;
using Osysharp.Notifications;
using NotificationsKitTests;

[Test(Studio)]
void a_browser_its_push_service_says_is_gone_is_dropped() {
  Http.Post.Stub((url, body, headers) => new HttpResponse { StatusCode = 410, Body = "" });
  runas (Bo) { SubscribeToPush(BoPhone); }
  runas (Ada) { AskForARead(DraftNamed("Spring issue"), PersonAt("bo@studio.test")); }
  Background.Settle();
  runas (Bo) {
    Assert.Equal(0, PushSubscription.Count());
    Assert.Equal(NotificationSendState.Sent, NotificationSend.Single().State);
    // Nobody else's: Cy cannot add a browser for Bo, nor see Bo's.
  }
  runas (Cy) {
    var bo = PersonAt("bo@studio.test");
    Assert.Denied(() => new PushSubscription { Person = bo, EndpointHash = "x", SealedEndpoint = "x", Host = "x" });
    Assert.Equal(0, NotificationSend.Count());
  }
}
```

From [`tests/push.test.osy`](tests/push.test.osy).

## Learn more

- [The Osysharp.Notifications reference](https://osysharp.com/reference/ui/notifications-kit/)
- [Osysharp.Notifications in the package catalogue](https://osysharp.com/packages/#osysharp-notifications)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
