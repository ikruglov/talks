These are the slides for my talk "Sereal and its tooling", delivered at [YAPC::EU 2015](http://act.yapc.eu/ye2015/talk/6350).

To create the slides I used [RemarkJS](http://remarkjs.com), which turns
markdown into a nice HTML/JS slideshow.

To view the slides, you'll need something that can serve the contents of the
directory over http.  If you don't have one, you can install the one I used:

    go get github.com/dgryski/trifles/servedir
    go install github.com/dgryski/trifles/servedir
    ./servedir

Then browse to http://localhost:8080/slides.html

Also, you can simply open [slides.md](https://github.com/ikruglov/talks/blob/master/sereal-and-its-tooling/slides.md) (limited view).

ACKNOWLEDGMENT

The framework for this slides was kindly borrowed from Damian Gryski.
