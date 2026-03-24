# Rotary Phone Project

This is an account of a project I recently did. The files aren't intended to constitute a cohesive runnable project, they're just a loose collection of scripts. It's mostly here to remind me how it works, but hopefully it's useful to others.

## Background

I was inspired by this [project](https://github.com/mnutt/rotary) and decided to follow their instructions in order to build a similar setup for my niece. I struggled to get incoming calls working for many months. My goal here is to provide consequential details that were ommitted from the original. 

## Features

Some things I wanted it to be able to do:

* Dial out to a short list of family contacts. It's not something I think about much, but when I was a kid there was a phone on the wall and once I could reach it, I could use it. Now, if you're not old enough to have a cell phone, you also can't call anyone at all.
* Allow incoming calls from that same list of family contacts. Requires an actual phone number.

### Features that differ from original

* No MTA train status service
* WiFi bridge with Raspberry Pi that provides phone location flexibility
* Missed call email notifications

## Equipment

60 years ago, every household in America had one or more rotary phones, so they're not exactly hard to find. I bought one off of Ebay for ~$25. Not all of them are guaranteed to be in working order, but these things are pretty solid so if the outside looks undamaged, it's likely alright. Worst case you might have to spend another $25 and try again. I got a [Western Electric Bell 500](https://en.wikipedia.org/wiki/Model_500_telephone).

If we lived in 1960 (or even 2000) I could plug it into the wall and the project would be mostly done. But if I want to do anything interesting, I need to adapt it to VoIP. My initial thought was to hack up the inside of the phone and use a Raspberry Pi or ESP32 or something, but between driving 48V DC or replacing most of the internals it seemed a bit out of reach. Instead, I used the phone unmodified and connected it to a [Grandstream GS-HT802](https://www.grandstream.com/products/gateways-and-atas/analog-telephone-adaptors/product/ht802) which lists rotary support in the specs. This currently goes for ~$50 [on Amazon](https://www.amazon.com/Grandstream-GS-HT802-Analog-Telephone-Adapter/dp/B01JH7MYKA).

To accomplish the other features, I wanted to use Asterisk, an open source PBX. Using Asterisk to make a phone that tells you jokes is a bit like using Kubernetes to host your blog but it seemed to do everything I wanted. I purchased a Raspberry Pi 5 with 4gb of memory. I purchased a bundle from a local retailer which included an SD card, a case, power adapter, and a few other things. You can likely set this up on a lower-end or older version of a Raspberry Pi. 

![Diagram of equipment setup](./images/diagram.png)

## Rotary Phone Setup

Pretty much just plug the RJ11 port of the phone into port 1 on the GrandStream. This should mostly Just Work, but there are some potential caveats depending on the rotary phone you receive. I'd try it first, but if you have trouble with it ringing later in the process you can try:

* It's possible that your phone has been wired for a "party line" instead of regular service. This would require unscrewing the cover (make sure it's unplugged from the Grandstream) and a small amount of rewiring which differs based on your phone model.
* The one I received _wasn't_ wired for a party line, but was effectively muted. I had to take the cover off and unhook part of the ringer. I haven't seen discussion of this online anywhere.

Opening it up:

![Inside the phone](./images/inside.jpg)

After everything else checked out, I realized the ringer had been locked, and moved it to this position to fix:

![Mute lever](./images/rotary_mute.jpg)

## Raspberry Pi Setup

I installed Raspberry Pi OS using the imager provided on the [Raspberry PI website](https://www.raspberrypi.com/software/). 

## Asterisk setup

I installed Asterisk on Rasperry Pi OS using [these instructions](https://raspberrytips.com/install-asterisk-on-raspberry-pi/). Asterisk is trying to move from legacy `sip` module to `pjsip`, but I was only able to get `sip` to work so I stuck with that. The `/etc/asterisk/sip.conf` relevant parts:

```
[rotary]
type=friend
nat=force_rport,comedia
secret=s3cr3t
host=dynamic
context=rotary-context
qualify=yes
```

The `nat` and `host=dynamic` aspects wouldn't be necessary if the Grandstream was on the same network as Asterisk. 

## Connecting the phone to Asterisk

Plugging the Grandstream device in, it automatically obtained an IP via DHCP and I was able to log into its web interface using default credentials. Setup was pleasantly straightforward, I just configured fx-1's sip server to point to the Asterisk server's address.

Relevant settings from `fxs-1`:

```
Active: Yes
Primary SIP Server: [asterisk ip]
NAT Traversal: Keep-Alive *
SIP User ID: rotary
SIP Authenticate ID: rotary
SIP Registration: Yes
Enable SIP OPTIONS/Notify Keep-Alive: OPTIONS *
Enable Pulse Dialing: Yes
Enable Hook Flash: Yes
Enable High Ring Power: Yes **

* - only required because of the RPi wifi bridge
** - maybe not necessary, but makes ringer more likely to work
```

At this point, you should be able to pick up the phone's handset and get a dialtone. If you don't, there are some different debugging avenues:

### Asterisk side

You can run `sudo asterisk -rvvvvv` to start the asterisk repl. Then:

* `sip show peers` should list `rotary/rotary` with the right Host and hopefully a Status of "OK"
* `sip set debug on` will give loads of SIP debug data, after enabling that you can unplug and replug the grandstream and hopefully see it trying to connect

### Grandstream side

If you're not seeing any peers or any sip logs from asterisk, you can also use the Grandstream web UI and enable sip logging to hopefully see what is going on.


## Using Raspberry Pi as a WiFi bridge

This allows you to place your phone and Grandstream anywhere within WiFi range instead of having to plug the Grandstream to your router or access point via ethernet. But I recommend following the setup without the bridge first as it makes it much easier to debug, and only after you have both outgoing and incoming calls fully working, set up the WiFi bridge. Instructions coming soon. 


## Building Stuff in Asterisk

At this point the phone is connecting to Asterisk and the real fun can begin.

### Play a song

The easiest one first. Asterisk can play back an audio file, but is relatively limited in the types of files it can play. You'll likely need to convert your file to 8KHz wav. Then put something like this in `/etc/asterisk/extensions.conf`:

```
[rotary-context]
exten => 1234,1,Answer()
    same => n,Playback(/path/to/file) ; Must be in /var/lib/asterisk/sounds/en/, do not specify file extension here
    same => n,Hangup()
```

You can reload Asterisk config with (among others) `sudo systemctl reload asterisk`. This will let you dial 1,2,3,4 on the phone, answer, play the audio, and hang up on you. One detail ommitted from the original writeup is that you have to place audio files in `/var/lib/asterisk/sounds/en/`, or at least I did in my setup.

### Phone Calls

At this point many hours in, we're almost to the part you could have achieved 30 years ago by buying a phone and plugging it in to any house in America.

In order for someone to use the phone to talk to another person with a regular phone, we need to connect to the phone network. Long ago it was apparently possible to use Google Voice, but from everything I've read it's no longer possible. I signed up for Twilio: not the cheapest, but it's reputable and developer-focused so pretty easy to work with. Buying a number will cost a few dollars a month.

Twilio offers two different services that seem like they could be relevant: I started out exploring "Twilio Elastic Sip Trunking" which "enables you to make & receive telephone calls from your IP communications infrastructure around the globe over a public or private connection" and sounds like exactly what I need. This is probably the "correct" way to bridge Asterisk into Twilio, especially if you're doing something real. But this approach assumes your Asterisk server has a public IP and mine sits behind my home router. Using this approach I was able to have Asterisk dial out, but Twilio could not make calls into Asterisk. NAT strikes again!

Instead, I went the route of creating a Programmable SIP Domain in Twilio and having Asterisk just act like another SIP client. This let Asterisk register with Twilio and punch through NAT using the usual OPTIONS keepalive trick.

The steps are:

1. Create a [SIP Domain](https://console.twilio.com/us1/develop/voice/manage/sip-domains)
2. Create a Credential List. This is how Asterisk will authenticate with Twilio. Set up Voice Authentication with the Credential List.
3. Enable SIP Registration.

On the Asterisk side, we'll need something like this in our `/etc/asterisk/sip.conf`, note that the order of these matters:

```
[general]
minexpiry=605
defaultexpiry=605
qualify=yes
qualifyfreq=20
register =>replace-with-your-sip-name:replace-with-your-sip-password@yourSipSubdomain.sip.us1.twilio.com

[twilio-trunk](!)
type=peer
context=from-twilio ; Which dialplan to use for incoming calls
dtmfmode=rfc4733
canreinivite=no
insecure=port,invite

[twilio](twilio-trunk)
host=replace-with-your-sip-name.sip.us1.twilio.com
defaultuser=yourUserName        ; username
remotesecret=totallysecure ; password
defaultexpiry=605
minexpiry=605
qualify=yes
```

Replace the host with the host from the SIP Domain you created in Twilio. And replace the username and password values with the username and password from the Credentials List you created.

* `qualify=yes` will cause Asterisk to keep the NAT hole alive by periodically sending OPTIONS requests. 
* `defaultexpiry` and `minexpiry` keep us from sending so many pings that Twilio rate-limits us. I found that these have to be specified under `[general]` above all else
* `context=from-twilio` means that incoming calls will wind up in the `from-twilio` context in `extensions.conf`, where you can route them to your rotary phone:

After you reload Asterisk, you can run `sudo asterisk -rvvvvv` and `show sip peers` and see both your rotary phone as well as Twilio:

```
CLI> sip show peers
Name/username             Host                                    Dyn Forcerport Comedia    ACL Port     Status      Description
rotary/rotary             192.168.1.33                             D  Yes        Yes            5060     OK (22 ms)
twilio/yourUserName       54.172.60.0                                 Yes        No             5060     OK (18 ms)
```

#### Dialing out

With the connection established, we need to tell Asterisk how to route a call to Twilio, and need to tell Twilio what to do when it gets it.

Asterisk config, in `extensions.conf`:

```
exten => 81,1,Answer() ; Call Dad
 same => n,GoSub(speak,s,1("Calling dad\!\!"))
 same => n,Set(CALLERID(all)="Rotary"<19787777777>)
 same => n,Dial(SIP/twilio/+12565121024)
```

In this case you'd replace `19787777777` with the Twilio phone number you purchased, and `12565121024` with the number you're calling. I'm not entirely sure the caller ID part is necessary, but it seems like it.

Back in Twilio, we want our SIP domain to receive a call and pass it right on through via our purchased phone number. To do this we can create a [TwiML bin](https://console.twilio.com/us1/develop/twiml-bins/twiml-bins):

```
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Dial answerOnBridge="true" callerId="{{#e164}}{{From}}{{/e164}}">{{#e164}}{{To}}{{/e164}}</Dial>
</Response>
```

Give it a name like "OutboundCall", then go back to the SIP Domain configuration:
* Set Call Control Configuration to configure with "Webhooks, TwiML Bins, Functions, Studio, Proxy"
* A call comes in: set to "TwiML Bin" then select your bin name
* The rest of the fields don't matter

Make sure SIP routing is Active.

Now try dialing "81" from your rotary phone. It should route to your recipient.

I went back and forth a bit on setting up emergency service. (911, in the US) If you do this, you'll absolutely want to make sure your home address in Twilio is accurate. I haven't set up emergency dialing yet because while I can impress on my son the serious nature of calling 911, his friends might treat it more like a toy. Still, something worth considering.

#### Receiving calls

With our outbound setup, Asterisk could technically dial _any_ number but we limit which numbers can actually be dialed by only explicitly dialing them in `extensions.conf`. For receiving calls, we want to make sure we also limit it to specific callers. We could easily achieve "pass any call through" with the same TwiML Bin approach, but "limit to phone numbers x,y,z" is a bit too much for TwiML so I opted for a Twilio Function.

In Twilio, [create a Service](https://console.twilio.com/us1/develop/functions/services) to make a new function. I named mine `incoming-call`, with this logic:

```javascript
const allowed = ['+12565121024', '+13435555555', ...];

exports.handler = function(context, event, callback) {
    console.log("Receiving call", event.From);
    let twiml = new Twilio.twiml.VoiceResponse();

    if(allowed.includes(event.From)) { // Ensure the number is in E.164 format
        console.log("Matched, dialing...");
        const dial = twiml.dial({answerOnBridge: true}).sip("rotary@replace-with-your-sip-name.sip.twilio.com");
    } else {
        console.log("Call was rejected");
        twiml.reject(); // Rejects the call if the number does not match
    }

    callback(null, twiml);
};
```

Make sure you save and click "Deploy all" to actually deploy it.

Then go to Phone Numbers -> Manage -> Active Phone Numbers and select the number you purchased to configure it. Choose "Webhooks, TwiML Bin, Function, Studio Flow, Proxy Service", then choose "Function" and the one you named `incoming-call` and choose the Environment and Function Path. (there should be only one option for each)

At this point, you can go back to the Function editor, enable Live Logs, and actually try to call your Twilio number and see it route the call. It'll send it on to Asterisk, who will probably immediately dump it since you haven't configured anywhere for it to go. We can do that in `extensions.conf`:

```
[from-twilio]
exten => s,1,NoOp(Incoming Call)
 same => n,Dial(SIP/rotary)
 same => n,Hangup()
```

This is where you could put other fancy logic, for instance if you wanted to let other people call you and hear your songs or something.

At this point you should be able to dial your Twilio number from your cell phone and, with a great deal of luck, your rotary phone will ring.

## Conclusion

I built two of these setups, one for my niece, and one for myself. I love getting calls from my niece out of the blue, and I love seeing her excitement whenever she hears her phone ring.
