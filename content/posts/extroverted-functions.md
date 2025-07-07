---
title: "Shy Code"
date: 2025-07-07
summary: "An antipattern I see often in intertwined code"
draft: false
---
Let me preface by saying that I am about to say is nothing new. All these ideas are decades old and people much smarter than me have stumbled upon or deduced this ideas. Now, I have been writing Go heavy code for about 4 years and having seen some good and a lot of bad codebases, an analogy that struck me today was how a lot of the code I call bad is just "introverted". 

Look at this function signature from a real codebase:
```go
func (c *Controller) addHostsFileDNSEntries(
	ctx context.Context, node *Node) error {...}
```
What's going on here?
1. What hosts file are we talking about? Do you know the path to it? How do I read and write into it?
2. Which DNS entries??

One might say, "this is picked out of context" and you will have to take my word for it when I tell you this is from a codebase I am quite intimately familiar with and I was still stumped by this signature and the 100 line function that followed. Concisely put, it takes almost nothing as input, considers information within its state *and implicit knowledge* about 
1. Where the hosts file is,
2. Which DNS entries to construct and 
3. How to talk to the node

Where this comes to bite you in the ass is when you want to either reuse or test this code. You can't tell it what to do! It becomes a black box with barely a knob on it and a sticker that says "Use Me". For what?? This is the opposite of the toy where [everything fits into the square hole](https://youtube.com/clip/Ugkx0oLjsA77EKwgrk_ObqT-6AFHb7kZ_GIs?si=qjK5wa8dml3LDWii), this is a calligraphic cutout in the box.

It is a *shy function*. It feels awkward in asking the caller for **any** details. `Umm, 😅, I guess I'll figure it out` ass code.

If one looks at this from the lens of Conway's Law, one might see that this is also just a reflection of the culture of the team and the interactions not *across* teams but *within* the team itself. There is friction in asking someone what the exact requirements are, if they might change and in what way. It seems easier to some to write code in a way that **exactly** fixes the current bug, and nothing more.

But anyone can rant, how would the great *I* write it? Well, I would not write this function at all, I would break it such:
```go
type HostsFileEntry struct {
    IP net.IP
    Hostnames []string
    Comment string
}

// To store an in mem representation of the contents
type HostsFile struct {...}
func (*HostsFile) parse(io.Reader) error
func (*HostsFile) AddEntries([]HostFileEntry) error
func (*HostsFile) write(io.Writer) (int, error)
func (*HostsFile) Bytes() []byte
func NewFromFS(fs afero.Fs, path string) (*HostsFile, error)
func NewFromBytes([]byte) (*HostsFile, error)
func FlushToFS(hf *HostsFile, fs afero.FS, path string) error

```

You don't even have to write out this code anymore, the hard (and fun) part of ideating the structure is done. Gemini and Claude will have a field day with this. The thing about good code is, like how a good character is recognizable from its silhouette, you can immediately see how the pieces fit together. 

```go
// Do not be shy!
// Let the caller figure out where the file is on which FS and what entries need to be added.
func AddDNSEntriesToHostsFile(fs afero.Fs, path string, entries []HostsFileEntry) error {
    ...
    hf, _ := hostsfile.NewFromFS(fs, consts.HostFilePath)
    _ = hf.AddEntries(entries)
    _ = hostsfile.FlushToFS(hf, fs, path)
    ...
}
```
In fact, I would go ahead and *delete* this function entirely. It gives me barely anything over just doing it in the caller. That is also what is afforded to us by good API design of the hostsfile package.

There ends my contrived example that makes this code a bit easier on the eyes and much more testable. 
