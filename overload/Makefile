
flags = -s -S -V geometry:margin=1.5in

fn.pdf: fn.md
	pandoc ${flags} fn.md -o fn.pdf

all: fn.pdf Makefile

clean:
	rm -f fn.pdf
