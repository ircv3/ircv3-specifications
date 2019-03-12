# How to write an IRCv3 Specification
This rough guide aims to take you through the process of writing an IRCv3 specification and getting it accepted! It doesn't lay out exactly what you **must** do for a document to get accepted, but should be a useful set of suggestions and considerations to keep in mind as you go through the process.


## Why write a spec?
There are a few main reasons why you'd want to write up a spec for inclusion within the IRCv3 Working Group:

- You'd like to change an aspect of the IRC protocol, and write down in **very exact detail** how that change works.
- You have a cool, interesting, and/or useful feature in your software and think that other software should include it.


## What you should know before writing a spec
Here are some things to keep in mind if you intend to write up a specification and take it through the IRCv3 process:

**The spec needs to lay out every single detail of how the feature works.** Specifications that we adopt don't really leave anything to chance. Hopefully, at least. Before a spec is accepted, that document will need to go over all the aspects of the feature and how they work in great detail, even to the point of being tedius and boring.

**The spec will be picked apart line-by-line, paragraph-by-paragraph, and section-by-section.** Since these documents tell developers how to write features in software, it's really important for them to be complete and easily-understandable, while keeping it as short and simple as possible. It's a difficult balance to find, and we're pretty ruthless about editing specs to get them there before we accept them.

**Getting it accepted may take a while.** Even once something looks good on paper, there can be a bit of a delay before we merge it in. This can be for many reasons, but usually it's that we're waiting for input from a particular type of implementer (asking a major server developer whether it looks workable), waiting for a/more test implementations, or something similar.

**Spec talk can get a bit heated.** Especially if a specification has had a lot of back-and-forth, people can get a bit less-careful with their words and angry while discussing it. We do our best to keep things calm and keep discussion constructive, though.

Getting a complete specification to these standards is difficult. It can be annoying and irritating, and can feel like your specification is being nit-picked over, particularly when lots of eyes are reviewing it. It's good to keep these points in mind, and do your best to not let them get to you. If you feel like certain interactions are unwarranted or don't follow our code of conduct, [this page](https://ircv3.net/conduct.html) lists the project moderators who you can contact.

Now that we're through those points, let's go through how writing a spec works.


## Spec Stages
Writing a specification has a few typical stages. This section lays out the ones a feature tends to go through, different approaches for them, and suggestions for how to move to the next stage.

### The initial idea
Usually, a spec idea comes from one of two places, either:

- Someone just has an idea for a new feature, method, etc.
- Someone wants to write up a behaviour that implementations in the wild are already doing (and maybe have been doing for a long time).

For an entirely new idea, there's typically lots of specifics to work out. For example, an idea like _"Allow UTF-8 channel/nick names"_ sounds reasonable on the surface. But there are a lot of follow-up questions to be worked out (such as: _"How does it interact with clients that don't support UTF-8?"_, _"How are they encoded, if they should be presented to clients that don't support UTF-8?"_, _"How do you prevent Unicode 'confusables', or allow servers to prevent confusing names?"_, etc). This is where the step below (Working out specifics) can be useful.

For an existing behaviour, there's typically not many questions to answer (or not as many). For most behaviour, fall back to what existing implementations do. Maybe create a new capability or `RPL_ISUPPORT` token so that servers can indicate they follow the specification exactly, etc. Most of these go straight to the spec-writing stage.

### Working out specifics
This is where the bulk of decisions are made on how the new feature/behaviour works. This is typically done in a few different ways:

- Creating an issue on the [ircv3-ideas repo](https://github.com/ircv3/ircv3-ideas) where all sorts of proposals, feedback, and ideas about how the feature should work can be discussed back and forth.
- Discussing the idea in the [`#ircv3` channel](https://ircv3.net/contact.html) directly.
- The spec author doing their own research, testing, and/or thinking about how it should work.

The most immediate of these is just opening a discussion in our IRC channel. However, these freeform discussions can become not-very-useful arguments if things aren't kept on track and people don't move on from a point when there's a stalemate. In addition, without anyone recording thoughts, decisions, or ideas down to a more permanent place (such as that repo issue), talking points can very quickly become lost or forgotten about.

An issue on the ideas repo is the most permanent discussion channel we have. It's a very useful tool for talking out different ideas and aspects, and in particular allows people to write down thoughts and reasonings in a longer form than immediate in-channel discussion provides for. With the same caveats of avoiding arguments above, this can be a very useful place to work out specifics.

And self-research, testing, and consideration from the proposer (and/or spec writer). This can be a very useful tool, but thoughts gained from this often need to be explained. After doing some of this, I tend to add a comment to the issue which goes through what I've found, and why my decision or preference falls the way it does.

### Writing the spec
After you have an idea and the specifics of that idea worked out, it's time to write up an initial copy of the specification. Something which you can present to the IRCv3 WG as a whole for consideration and feedback.

Our [example-spec.md](./example-spec.md) document can be extremely useful here, giving you a rough template you can use to get started.

As you write up your specification, the main points which you want to emphasise are:

- Keep it as clear and simple as possible.
- Explain the feature in as much detail as you can. Software authors shouldn't have additional questions about how your feature works or how they should write it after reading the document.
- Don't repeat yourself unless you're stating a different point, or a different aspect of the same point.
- As the writer of the specification, this is where you're making the decisions on how the feature/behaviour works. If one was created, look back on the ideas repo issue, or on the in-channel discussion that happened around the idea, and keep it in mind as you make your decisions.

Specifications are written in a very specific way. Very matter-of-fact, focusing on what implementations (clients, servers, bots, etc) should do and how they should do it. Take a look at our existing specs for reference, at the IETF [Guide for Internet Standards Writers](https://tools.ietf.org/html/rfc2360), and at the [Key Words RFC](https://tools.ietf.org/html/rfc2119).

Keep in mind that once submitted, others are going to comment on how the wording, on the structure of the document, and make very exact, specific, and general change suggestions. There isn't a single IRCv3 spec that hasn't had a round of changes, so don't feel worried or singled-out when your spec goes through this same process. It's a good idea to not be too precious about specific wording, because there will be changes.

Once you've written up your initial version, review it, make sure it goes over every point and explains things in detail. Writing an implementation of the feature based on your document can also help uncover places where more explanation or more specifics are required.

### Submitting the spec
Actually submitting the document is as simple as forking the [ircv3-specifications repo](https://github.com/ircv3/ircv3-specifications), making your changes, and then submitting a pull request. Make sure the pull request description gives a basic outline of how the feature works as you see it, and how it's useful to other IRC software.

### Editing the spec
This is the longest part of writing a spec. Once submitted, other implementers and members of the IRCv3 Working Group will make comments, discuss potential changes, and go back-and-forth about the specifics of the submitted document.

During the editing process, other members of the group will probably write up implementations of your submission, and come back with where they had trouble, changes they would recommend, and where things could be explained more simply or more clearly.

For a spec to move out of the editing stage and get accepted, it requires a few implementations, a good portion of the [technical board](https://ircv3.net/charter.html#technical-board) being on-board with it, and for remaining discussions or questions about the document to be answered and resolved.

### Accepted or rejected
And the result is that either the specification will be accepted, or that it will be rejected.

If your spec is accepted, other implementers will be able to write draft implementations of it and feed-back on it. If necessary changes will be made to clarify points of the spec, and/or to resolve issues that are found with a wider pool of implementations. Once we're happy with it, it will be moved out of draft status and accepted completely.

If your spec is rejected, we will give some feedback as to why. It's good to keep this feedback in mind if you intend to rewrite or otherwise edit and re-submit your proposal at some point in the future. This may also be a good time to let a different member of the group that agrees with your proposal take it on themselves, letting them change it up and propose it instead.
