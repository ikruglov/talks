These are the slides for my talk "Optimize Sereal", delivered at September Go meetup, Amsterdam 2014

To create the slides I used [RemarkJS](http://remarkjs.com), which turns
markdown into a nice HTML/JS slideshow.

To view the slides, you'll need something that can serve the contents of the
directory over http.  If you don't have one, you can install the one I used:

    go get github.com/dgryski/trifles/servedir
    go install github.com/dgryski/trifles/servedir
    ./servedir

Then browse to http://localhost:8080/slides.html

Also, you can simply open [slides.md](https://github.com/ikruglov/talks/blob/master/go-sereal-optimization/slides.md) (limited view).

ACKNOWLEDGMENT

The framework for this slides was kindly borrowed from Damian Gryski.
