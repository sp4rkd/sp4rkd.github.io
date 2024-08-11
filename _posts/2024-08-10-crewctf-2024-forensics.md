---
layout: post
title: CrewCTF 2024 forensics
subtitle: The harsh reality of USB pcaps
gh-repo: sp4rkd/CrewCTF2024-forensics
tags: [ctf, forensics, usb, proxmark3, mifare, felica, writeup]
cover-img: /assets/img/crewctf/2024/bg.png
thumbnail-img: /assets/img/crewctf/2024/thumb.png
share-img: /assets/img/crewctf/2024/bg.png
---

# Crew CTF Writeup
{: .box-success}
I have to be honest, the set of challenges was exceptionally interesting with reasonable difficulty progression. What made me sad though is the lack of recognition for the last forensics challenge (Unfare) and I thought it definitely deserves some recognition and archiving. I highly recommend checking out write-ups for different challenges as well; @, @, and @ produced tremendous pieces themselves.

As always, without further ado, here go my notes.

## Recursion

Initially, you face the eternally conflicting decision: to click on anything related to recursion or to avoid a headache. Luckily, this challenge was not as terrifying as it originally seemed. 

### Overview of the Challenge
The provided pcap traffic sets the tone for this forensics weekend, featuring the Universal Serial Bus, also known to us commonwealth pigs as USB. As you dig through the traffic, all you see is the usual chaos that might or might not lead somewhere. In this or any other case, you also want to see other tools chew on the interesting parts to either support or disprove your ideas while you scream into a pillow. 

Let’s take a stroll with our good old pal, binwalk, first.

![Image](/assets/img/crewctf/2024/binwalk_default.png)

### Is This Actually an Embedded Something?

For those that didn't ever think of throwing binwalk at a pcap before, notice it can extract quite painlessly too. Upon carving open the mysterious compressed file, we get hit with a first whiff of recursion - it's another pcap. 

How intriguing! Now, wait—does this pcap also contain a whole lot of something, primarily an embedded 7z?

How intriguing! Now, wait—does this pcap also contain a whole lot of something, primarily an embedded tar?

How intriguing! Now, wait—does this pcap also contain a whole lot of something, primarily an embedded zip?

![Image of Solve](/assets/img/crewctf/2024/matrioshka_debulk.png)

Are you, by chance, a pleasure model recursion?

## Unfare

Buckle up and say <em>**fare**</em>well to your sanity because this one felt special. During the unavoidable initial service overload of the CTF infrastructure, I managed to slip through and pull only one challenge before the crash. What do you know? It ended up being the epitome of chasing the dragon for the night—the Un<em>**fare**</em>. 

I just couldn’t stop shooting to the point where sleep was no longer an option. Waking up from an involuntary micronap at 4:43 AM to the glow of terminals plastered with hellhole packet extractions was truly a moment for self-reflection. Almost 600 teams have tried, 11 have solved (some more janky than others), but a whole lot of people were so close, yet still too damn far… and I was one of them. Nonetheless, pretty impressive numbers for a pcap challenge, so massive shoutout to the designer (@sealldev).

![Discord is mostly a support group anyway](/assets/img/crewctf/2024/discord_yet_so_far.png)

### Overview of the Challenge

You are provided with a single pcap file containing USB communication from the host that sniffed the traffic and the following description.

Someone was reading cards on my computer! Lucky I was listening...
{: .box-error}

 There are four different addresses in play, all sharing the same prefix (**10.4.X**). Some traffic seems to be paired, given the sequencing, which might indicate some input/output distinction; other packets are just there for you to paint the whole picture. Digging through the recognizable patterns in the transmitted leftover data, we can see Proxmark3 (**PM3**) signatures in both bootrom and ncd files.

![Felica and PM3 files](/assets/img/crewctf/2024/info_dump.png)

As well as artifacts from the Felica manufacturer and a majority of URB bulks that are somewhat predictable. Thus, the general focus of this challenge becomes clearer.

![Sony FeliCa](/assets/img/crewctf/2024/felica.png)

### Data Extraction

After extracting leftover data from the source and destination addresses **10.4.1** and **10.4.2** (due to the PM3 prefix), you can see slight deviations in packet structure but a similar theme. In all fairness, the opportunity to finally explore PM3 and different USB tooling in depth was a good hit of happy chemicals on its own, so I went ahead and explored.

For wireshark, I generally used this view layout and filter baseline:

```plaintext
!(usb.urb_type == URB_SUBMIT && usb.endpoint_address.direction == IN) && 
!(usb.urb_type == URB_COMPLETE && usb.endpoint_address.direction == OUT) &&  
(usb.src == "1.4.2" || usb.dst == "1.4.2") &&
usb.capdata == ""
```

![Wireshark view](/assets/img/crewctf/2024/wireshark_view.png)

![Patterns in leftovers](/assets/img/crewctf/2024/leftover_dissection.png)

Making different data formatting options greatly helps during processing. Focus on essential sections, different formats, segment sizes, delimiters, and decoders to feed all you need. Tshark is your friend.


### Burnout

Now, about that hellhole. I would like to thank the main crutch of this violent night: the one and only CyberChef for always being there when I'm too burnt to handle. Trying to make sense of the data is an unhealthy obsession. The math jungle and its offsets can be approached in multiple ways, and even though the aforementioned tools were amazing, they didn't get me much further (at least they got me this random PSK squeeze).

![PSK squeeze](/assets/img/crewctf/2024/psk_clock.png){: .mx-auto.d-block :}

Coincidentally enough, the technical specification for Felica overlaps with something I noticed during the PM3 deepdive.

![Image of Felica specs](/assets/img/crewctf/2024/felica_transmissions.png)

To our collectively great demise, the entire card system flavor is a wrapper around the Mi**<em>fare</em>** model (hence some previous Mifare traces).

![Image of Mifare specs](/assets/img/crewctf/2024/mifare_transmissions.png)


Since I built the Proxmark3 toolkit before and fed it a bunch of different traffic, I was able to find relevant functions already in my scope ([Proxmark3 Mifare Commands](https://github.com/RfidResearchGroup/proxmark3/blob/master/armsrc/mifarecmd.c)) that would serve as the baseline for writing a reliable flag extractor.

The flag extractor on its own was expected, but there aren't many other sources, so the mifarecmd is almost the only reasonable reference. Even though it's still not the most casual read you've ever had in your life, it seemed like the only way to correct my somewhat wild data parsing process.

![Example crop of the 3000 lines worth of mifarecmd](/assets/img/crewctf/2024/casual_read.png)

## Conclusion

Believe it or not, it's still in the forensics category for a reason. Always remember the age-old mantra:
~~~
Doing the right protocols, 
   in the right frame, 
   IN THE RIGHT FORMAT 
         AND DATA TYPE 
gets_you{mad_cheeks}.
~~~

In reality, this challenge mainly requires you to recognize the traffic, decode, and concatenate a specific segment from both message types. So technically, everything else could be considered bloat. To prove my point about approaching the jungle in multiple ways, here is an honorary mention for @Ske and his absolutely beautiful, peak forensics performance. Because, in the end, the scoreboard doesn't care if your bloated script can manipulate the entire principle. The key is to give a shit, aim, and hit.

![Image of guessrensics](/assets/img/crewctf/2024/peak_guessrensics.png)

Anyway, here is my bloated function with improved packet parsing, thanks to the teammate of @.

{% highlight Python linenos %}
def parse_packets(filename):

   packets = open(filename, 'rb').readlines()
   packets = [x.strip() for x in packets]

   print ("# Parsing packets from %s", filename)

   block_index = 0

   for packet in packets:
      packet = packet.decode('utf-8')
      packet_info = {}
      print ("\n# Processing packet...")
      print (packet)

      data = bytes.fromhex(packet)
      offsetter = 0
      type_input = False
   
      # (bytes)Magic [0, 4]
      section_size = offsetter + 4
      packet_info['magic'] = data[:section_size]
      offsetter = section_size

      if (packet_info['magic'] == b'PM3a'):
         type_input = True

      # Safety check because I should've gone to bed 6 hours ago
      else:
         if (packet_info['magic'] != b'PM3b'):
            print("[magic] Skip...")
            continue

      print ("# isInput() => ", type_input)
   
      # (int)Length [4, 6]
      section_size = offsetter + 2
      length_ng = int.from_bytes(data[offsetter:section_size], 'little')
      offsetter = section_size
      # Woowoo masking
      packet_info['length'] = length_ng & 0x7FFF
      packet_info['ng'] = bool(length_ng & 0x8000)
      
      # Ouput packets specific
      # (int)Status [6, 8]
      if (type_input is False):
         section_size = offsetter + 2
         packet_info['status'] = int.from_bytes(data[offsetter:section_size], signed=True)
         offsetter = section_size
      
      # (int)Command [6, 8] ~ [8, 10]
      section_size = offsetter + 2
      packet_info['cmd'] = int.from_bytes(data[offsetter:section_size], 'little')
      offsetter = section_size
      #print ("[DBG] parsed cmd -> ", packet_info['cmd'])

      # (bytes)Data [8, 8+dl] ~ [10, 10+dl]
      section_size = offsetter + packet_info['length']
      packet_info['data'] = data[offsetter:section_size]
      offsetter = section_size
      
      # Read the input data header
      if (type_input is True):
         if (packet_info['cmd'] != 0x620): #'CMD_HF_MIFARE_READBL'
            print("[readbl] Skip...")
            continue
         packet_info['blockno'] = block_index = int.from_bytes(packet_info['data'][:1], 'little')
         packet_info['keytype'] = int.from_bytes(packet_info['data'][1:2], 'little')
         packet_info['key'] = packet_info['data'][2:8]

      # Output bytes are the actual ca$h
      else:
         flag[block_index] = chr(packet_info['data'][0])
         print("[DBG] F: ", flag[block_index])
      print("Block index -> ", block_index)

      # (hex~int)CRC [2+dl]
      packet_info['crc'] = hex(int.from_bytes(data[offsetter: offsetter + 2], 'little'))

flag = {}
parse_packets("/tmp/final_dump.txt")

flag = dict(sorted(flag.items()))
flag = ''.join(flag.values())
print("\n",flag)

{% endhighlight %}

Two days of joy condensed into a single image for your eyes only.

![Image of flag pop](/assets/img/crewctf/2024/flag_pop.png)

### Fun resources for anyone solving possibly related problem

![Where to find fpga hf](/assets/img/crewctf/2024/fpga_hf.v.png)
![Where to find fpga hi reader](/assets/img/crewctf/2024/fpga_hi_reader.v.png)
![Where to felica read write](/assets/img/crewctf/2024/felica_read_write.png)
![Where to find DES card mode](/assets/img/crewctf/2024/DES_card_mode.png)
![Where to find AES card mode](/assets/img/crewctf/2024/AES_card_mode.png)
