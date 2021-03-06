
Hi everyone, this is my first blog post, wow!

In this post I just wanna talk a little about the very minimalistic static blog system I ended up programming for this site.

I was trying with Jekyll before I decided to code my own, but as I already have most of my website already setted up, it ended up being a bit difficult integrating that with the Jekyll stuff. I then thought of maybe using something external like WordPress, but I then relized I prefer keeping everything in just one place.

So the final idea was to write a very simple and minimalistic program that would let me add blog posts as simple JSON metadata and MarkDown files and that it would compile this files into HTML pages.
An 80-lines Python script ended up doing all I needed, without having the hassle of learning Jekyll and making it integrate seemlessly with the site I already had, plus this little code is all mine meaning I can modify and add features fastly whenever I want.

Here is the snippet of the program that does all the magic:

	#!python
	from jinja2 import Environment, FileSystemLoader, select_autoescape
	import codecs
	import os
	import json
	import markdown

	class TemplateEngine:
	    def __init__(self):
	        self.env = Environment(
	            loader = FileSystemLoader("templates"),
	            autoescape = select_autoescape([])
	        )

	    def get_template(self, name):
	        return self.env.get_template(name)


	class PostLoader:
	    def __init__(self, folder):
	        self.folder = folder

	    def load_posts(self):
	        posts = []
	        for file in os.listdir(self.folder):
	            if not os.path.isfile(self.folder + "/" + file):
	                continue
	            with open(self.folder + "/" + file, "r") as stream:
	                post_data = json.load(stream)
	                posts.append(Post(post_data["title"], post_data["date"], post_data["content_file"], post_data["author"]))

	        return posts

	class Post:
	    def __init__(self, title, date, content_md_filename, author):
	        self.title = title
	        self.date = date
	        self.content_md_filename = content_md_filename
	        self.author = author
	        self.load_content()

	    def load_content(self):
	        content_md = ""
	        with codecs.open(self.content_md_filename, "r", "utf8") as md_file:
	            content_md = md_file.read()
	        self.content_html = markdown.markdown(content_md)

	    def get_link(self):
	        words = self.title.lower().split(" ")
	        return self.date + "-" + "-".join(words)

	class PostCompiler:
	    def __init__(self, folder, template_engine, post_template_name):
	        self.folder = folder
	        self.template_engine = template_engine
	        self.post_template = template_engine.get_template(post_template_name)

	    def compile_post(self, post):
	        with codecs.open(self.folder + "/" + post.get_link() + ".html", "w", "utf8") as output:
	            output.write(self.post_template.render(post = post))

	if __name__ == "__main__":
	    template_engine = TemplateEngine()
	    index_template = template_engine.get_template("index.html")

	    post_loader = PostLoader("_posts")
	    posts = post_loader.load_posts()

	    with codecs.open("index.html", "w", "utf8") as index_file:
	        index_file.write(index_template.render(posts = posts))

	    post_compiler = PostCompiler("posts", template_engine, "post.html")

	    for post in posts:
	        post_compiler.compile_post(post)



Every blog post is divided into a metadata part stored in a JSON file and the actual content file, stored as a MarkDown markup file. This makes post readable also on GitHub plus it makes it easy for me to write posts.

So I'm just a call on that Python little boy and some git magic away from making loads of cool posts!