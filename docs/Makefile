.PHONY: build serve

build:
	JEKYLL_ENV=production bundle exec jekyll build
	rm -rf docs
	cp -r _site docs

serve:
	bundle exec jekyll serve --watch
