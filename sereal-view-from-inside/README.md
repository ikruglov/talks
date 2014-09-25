These are the slides for my talk "Sereal: a view from inside", delivered as tech talk at Booking.com

To create the slides I used [RemarkJS](http://remarkjs.com), which turns
markdown into a nice HTML/JS slideshow.

To view the slides, you'll need something that can serve the contents of the
directory over http.  If you don't have one, you can install the one I used:

    go get github.com/dgryski/trifles/servedir
    go install github.com/dgryski/trifles/servedir
    ./servedir

Then browse to http://localhost:8080/slides.html

Also, you can simply open [slides.md](https://github.com/ikruglov/talks/blob/master/sereal-view-from-inside/slides.md) (limited view).

ACKNOWLEDGMENT

The framework for this slides was kindly borrowed from Damian Gryski.
