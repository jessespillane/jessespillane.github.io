# Bringing the Blog Back

I decided to bring back my blog. I pulled in old posts from various
incarnations of this website. I have a garbage bash script that
generates the blog automatically from md files. There is even an
[atom.xml](atom.xml) feed file, so that you subscribe in your favorite rss/feed
reader (if anyone besides me still does that).

I don't know why you'd want to use it (just use jekyll or hugo), but
you can take a look at it here: [generate_blog](generate_blog). Keep
in mind, it isn't well-written or effecient, but it does get the job
done.

To use, you are expected to update some of the variables at the top of
the script (like the name of the blog, among other things)

The script requires that you have the following installed:

- pandoc - to convert markdown to html
- tidy - to make html prettier

These are expected to be part of your path environment variable.

It expects there to be a 'header' and 'footer' file (representing your
html template). Blog entries are have the format
YYYY-MM-DD-whatever.md.  Normal pages are any other md files.

Your header should have {{$ title}} in the html title element so that
each page can replace it with the name of the page. 

You also need to place something like the following in your header:

~~~html

    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <link rel="alternate" type="application/atom+xml" href="http://www.jessespillane.com/atom.xml" />
    
~~~

...so that you:

- It's good to explicitly add the content-type. If you leave it out,
  you might find some odd formatting.
- You can tell browsers where the atom feed is (if your browser is set
  up to show that feed icon when a site provides it)

All md files are expected to be in the folder the script is run
from. Html files are generated in that folder as well.

The date for the blog is taken from the file name.  The title of the
page is the first top level heading (starting with #)

If you write in a file named draft.md, and run ./generate_blog, it
will move that draft to a blog post in the format
(date)-(lower_case_encoded_title).md.

Full script below:

~~~bash
#!/bin/bash
#This is garbage.  Don't use it.  Don't look at it. Like I said, it is garbage.
script_start=`date +%s`

#file name (sans extension)
blog_file_name="index"

#for page title and h1 heading
blog_title="Home"

#extension of input files
markup_ext="md"

#The markup processor we are using
markup_language="markdown"

#How many entries should be in the feed
feed_article_limit=10

#author's name
author="Jesse Spillane"

#rss feed title name
feed_title="Blog of $author"
feed_subtitle="An exploration of something or other"

#url (include ending forward slash)
url="https://www.jessespillane.com/"

#file name of the feed for syndication
feed_file_name="atom.xml"

#set time zone of your dates.
timezone_str="-05:00"

#remove the old html files/feed files
rm -f *.html
rm -f *.xml

#keep track of how many blogs we've come accross
blog_entry_count=0

post_heading="Blog"

# initialize feed
cur_date=$(date -u '+%Y-%m-%dT%H:%M:%SZ')

echo "<?xml version=\"1.0\" encoding=\"utf-8\"?>

<feed xmlns=\"http://www.w3.org/2005/Atom\">

	<title>$feed_title</title>
	<subtitle>$feed_subtitle</subtitle>
	<link href=\"${url}${feed_file_name}\" rel=\"self\" />
	<id>$url</id>
	<updated>$cur_date</updated>
" > "$feed_file_name"

header=$(<header)
footer=$(<footer)

if [ -f draft.md ]; then
    echo "Moving draft to new blog file."
    draft_contents=$(<draft.md)
    date_str=$(date +"%Y-%m-%d")
    title=$(grep -m1 "#" draft.md)

    if [[ $title == "" ]]; then
        echo "ERROR: File draft.md has no title"
        exit 1
    fi
    
    lowercase_encoded_title=$(echo ${title#\#} | sed -e 's/^[ \t]*//' | tr '[:upper:]' '[:lower:]' | sed -e 's/%/%25/g; s/ /-/g; s/:/%3B/g; s/\//%2F/g; s/#/%23/g; s/?/%3F/g; s/&/%24/g; s/@/%40/g; s/+/%2B/g;')
    new_file_name="${date_str}-${lowercase_encoded_title}.md"
    echo "new file name: $new_file_name"
    mv draft.md "$new_file_name"
fi

#start blog page:
echo -e "$header" > "${blog_file_name}.html"
echo "<h1>$blog_title</h1><h2>$post_heading</h2><p><a href=\"atom.xml\">Subscribe to Atom Feed<a></p>" >> "${blog_file_name}.html"

blog_month_year=""

#iterate over files in reverse order (for the sake of the blog ordering)
files=(*."${markup_ext}")
for ((i=${#files[@]}-1; i>=0; i--)); do
    
    markdown_file="${files[$i]}"
    html=$( pandoc -f "$markup_language" "$markdown_file" )

    #get first 10 characters of file name to see if it matches a date format
    date="${markdown_file:0:10}"
    
    is_blog=0

    #if the file format matches YYYY-MM-DD<whatever>.md, it is considered a blog
    if [[ "$date" =~ ^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[1-2][0-9]|3[0-1])$ ]]; then
        is_blog=1
        blog_entry_count=$((blog_entry_count+1))
    fi
    
    #get title from the first heading from markdown and strip the '#'
    #and leading whitespace
    title=$(grep -m1 "#" "$markdown_file")
    title=$(echo ${title#\#} | sed -e 's/^[ \t]*//')

    #If title doesn't exist, that is bad, so exit
    if [[ $title == "" ]]; then
        echo "ERROR: File $markdown_file has no title"
        exit 1
    fi

    filename="${markdown_file%.${markup_ext}}.html"
    
    #write the html into the template
    echo -e "$header" >> "$filename"
    echo "$html" >> "$filename"
    echo -e "$footer" >> "$filename"

    
    #replace title in template
    sed -i s/{{\$title}}/"$title"/ "$filename"

    #tidy up the output html to be nice looking
    tidy --indent yes --indent-spaces 2 --quiet yes --show-body-only auto --show-errors 0 --wrap 0 -m "$filename"
    
    #add to the blog list
    if [ $is_blog -eq 1 ]; then

        year="${markdown_file:0:4}"
        month="${markdown_file:5:2}"

        month_year_str=$(date -d "$date" "+%B %Y")

        if [[ "$month_year_str" != "$prev_month_year_str" ]]; then
            if [[ "$prev_month_year_str" != "" ]]; then
                echo "</table>" >> "${blog_file_name}.html"
            fi
            echo "<h3>$month_year_str</h3><table style=\"width: 100%\">" >> "${blog_file_name}.html"
        fi
        
        prev_month_year_str="$month_year_str"

        formatted_date=$(date -d "$date" "+%b %d %Y")


        encoded_filename=$(echo "$filename" |
                               sed -e 's/%/%25/g; s/ /%20/g; s/:/%3B/g; s/\//%2F/g; s/#/%23/g; s/?/%3F/g; s/&/%24/g; s/@/%40/g; s/+/%2B/g;')
        
        #add to blog list
        echo "<tr><td style=\"width: 130px\"><b>$formatted_date</b></td><td><a href=\"$encoded_filename\">$title</a></td></tr>" >> "${blog_file_name}.html"


        if [ $blog_entry_count -le $feed_article_limit ]; then
            escaped_html=$(echo "<![CDATA[${html}]]>")
            echo "
	<entry>
		<title>$title</title>
		<link href=\"${url}${encoded_filename}\" />
		<link rel=\"alternate\" type=\"text/html\" href=\"${url}${encoded_filename}\"/>
		<id>${url}${encoded_filename}</id>
		<updated>${date}T00:00:00$timezone_str</updated>
		<content type=\"html\">
 $escaped_html
		</content>
		<author>
			<name>$author</name>
		</author>
	</entry>" >> "$feed_file_name"

        fi
    fi
    echo "Wrote: $filename"
done

#generate the blog page
echo "</table>" >> "${blog_file_name}.html"
echo -e "$footer" >> "${blog_file_name}.html"

#replace the title in the blog page
sed -i s/{{\$title}}/"$blog_title"/ "${blog_file_name}.html"

#tidy up the html
tidy --indent yes --indent-spaces 2 --quiet yes --show-body-only auto --show-errors 0 --wrap 0 -m "${blog_file_name}.html"

echo "Wrote: ${blog_file_name}.html"



echo "
</feed>" >> "$feed_file_name"

echo "Wrote: $feed_file_name"

script_end=`date +%s`

runtime=$((script_end-script_start))
echo "Script ran in $runtime seconds"

~~~
