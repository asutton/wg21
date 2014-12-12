
file = delete

fn.html: ${file}.md head.html tail.html
	markdown ${file}.md > body.html
	cat head.html body.html tail.html > ${file}.html

all: ${file}.html

clean:
	rm -f ${file}.html
