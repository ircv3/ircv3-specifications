---
title: "`multi-prefix` Extension"
layout: spec
redirect_from:
  - /specs/extensions/multi-prefix-3.1.html
copyrights:
  -
    name: "William Pitcock"
    period: "2012"
    email: "nenolod@dereferenced.org"
---

## Description

When requested, the multi-prefix client capability will cause the IRC server to send
all possible prefixes which apply to a user in NAMES, WHO and WHOIS output.

These prefixes MUST be in order of 'rank', from highest to lowest.

Example:

    --> NAMES #tethys
    :hades.arpa 353 guest = #tethys :~&@%+aji &@Attila @+alyx +KindOne Argure
    :hades.arpa 366 guest #tethys :End of /NAMES list

    --> WHO #test
    :kenny.chatspike.net 352 guest #test grawity broken.symlink *.chatspike.net grawity H@%+ :0 Mantas M.
    :kenny.chatspike.net 315 guest #test :End of /WHO list

    --> WHOIS barmand
    :irc.chat 311 guest barmand barmand cosani.jp * :Armand
    :irc.chat 319 guest barmand :~&@%+#falco @+#raynor
    :irc.chat 312 guest barmand irc.chat :Antibes
    :irc.chat 318 guest barmand :End of /WHOIS list

## Errata

Previous versions of this spec did not specify that all possible prefixes which apply to users be also sent in WHOIS output. This was added for consistency with other replies that contain user prefixes.
