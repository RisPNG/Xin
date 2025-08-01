# Xin
A Packaged Package Manager TUI developed in Golang using Bubble Tea for Debian-based systems.

I plan to develop this because I like installing packages with `--no-install-recommends --no-install-suggests` in `apt` and `nala` or `-R` in `Aptitude`. It's bad in regular practice because recommends are somewhat crucial (not as crucial as depends).

This can be a good start to building out a completely minimal system without all the unnecessary bloat if the user knows that a certain feature of the app are almost never used.

Using `apt` as the main package manager for the example. As a start, the workflow will be something like this:

1. `apt xinstall vim` or `apt xin vim` will install vim and its dependency but doesn't install the recommends and suggests.
2. Xin will then open another prompt after installation has succeed for user to select which from the recommends and suggests list that the user would like to include it with. The options will be Install Everything, Install Selected, or Skip.
3. Any further installation, meaning for each of selected recommends and/or suggests will also go through Xin and the process repeats.

It's a simple process, but eventually Xin will be able to manage all recommends and suggests, as well as pointing out which packages recommends/suggests each of the packaged packages, and allowing installation, removal, and purge. 

Why Xin?

Well, I ran out of whatever I want to call it so Install and NONO (X) gave birth to Xin. I used to know someone that goes by Xin... ahhh they went quiet.
