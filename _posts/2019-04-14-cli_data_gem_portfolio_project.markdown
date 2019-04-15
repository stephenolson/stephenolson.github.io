---
layout: post
title:      "CLI Data Gem Portfolio Project"
date:       2019-04-15 02:01:10 +0000
permalink:  cli_data_gem_portfolio_project
---


Entering the Portfolio Project I had wavering levels of confidence in my abilities and interest in this program. Lessons made sense, but felt abstract. However after watching the video walkthrough of Avi building a CLI Gem called Daily Deal, I felt renewed.

<iframe width="560" height="315" src="https://www.youtube.com/embed/_lDExWIhYKI" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

In this video Avi describes coding the CLI first, hard-coding it the data, and making that work before you get to the hard work of scraping. By coding the CLI first you have *something*; it's no longer a blank canvas in front of you. It then almost becomes a design problem, which is something I understand.

So that's what I did.

The Project's parameters are simple:

1. provide a command line interface
2. the CLI provides access to data from a web page
3. the data must go at least one level deep

It was suggested we choose a subject we are passionate about, but I wanted to work with data thatfelt concrete. A list of things, and then another level deeper with information about those things. I choose to scrape the Billboard Hot 100. While I'm passionate about music, I can barely identify any of the songs on the chart. However the information is clear.

The songs on the chart have the following basic attributes: 

1. Rank (the songs ranking: 1 through 100)
2. Title (the songs title)
3. Artist (the songs artist)

And deeper down they have the following attributes

4. Last Week (the songs position on the chart the previous week)
5. Peak Position (the songs peak position on the chart)
6. Weeks On Chart (the number of weeks the song has been on the chart)
7. Lyrics (a url linking the the songs lyrics)
8. Awards (such as: Highest Ranking Debut, Biggest Gain In Airplay , etc)  

Inititally I worked on my CLI, welcoming the user, and providing access to hardcoded data.

my Scraper Class:
```
class BillboardHot100::Scraper

   def self.scrape_billboard
    songs = []
    index = Nokogiri::HTML(open("https://www.billboard.com/charts/hot-100"))
    index.css("div.chart-list-item").each do |song|
      song_info = {}
      song_info[:rank] = song.css(".chart-list-item__rank").text.strip
      song_info[:name] = song.css(".chart-list-item__title-text").text.strip
      song_info[:artist] = song.css(".chart-list-item__artist").text.strip
      song_info[:last_week] = song.css(".chart-list-item__last-week").text.strip
      song_info[:peak_position] = song.css(".chart-list-item__weeks-at-one").text.strip
      song_info[:weeks_on_chart] = song.css(".chart-list-item__weeks-on-chart").text.strip
      # song_info[:lyrics] = song.search("a.chart-list-item__lyrics").first.attr("href").strip

       songs << song_info
    end
    songs
  end

   def make_songs
    songs = BillboardHot100::Scraper.scrape_billboard
    BillboardHot100::Song.create(songs)
  end

 end
 

```
and my Song Class:
```
class BillboardHot100::Song

   attr_accessor :rank, :name, :artist, :last_week, :peak_position, :weeks_on_chart

   @@all = []

   def initialize(song_hash)
    song_hash.each {|key, value| self.send(("#{key}="), value)}
    @@all << self
  end

   def self.create(songs_complete)
    songs_complete.each do |song_hash|
      self.new(song_hash)
    end
  end

   def self.all
    @@all
  end

   def self.find(index)
    self.all[index-1]
  end

 end
```
As you can see I was using Hashes to hold the data, and while this was working, it wasn't what the project called for. Like I said though, it was working!

In the lessons, you are working through tests and when they're all passing that can feel rewarding, but building a useable program felt better than any previous coding I had done. But now I needed to rework my code...

Scraper Class:
```
class BillboardHot100::Scraper

  def get_page
    Nokogiri::HTML(open("https://www.billboard.com/charts/hot-100"))
  end

  def scrape_songs
    self.get_page.css("div.chart-list-item")
  end

  def make_songs
    scrape_songs.each do |song|
      BillboardHot100::Song.songs(song)
    end
  end

  def scrape_dates
    self.get_page.css("div.chart-detail-header__select-date")
  end

  def make_dates
    scrape_dates.each do |date|
      BillboardHot100::Date.dates(date)
    end
  end

  # billboard.com changed their code, making a separate "number_1" unecessary
  # def scrape_number_1
  #   self.get_page.css("div.chart-number-one__info")
  # end

  # def make_number_1
  #  scrape_number_1.each do |song|
  #    BillboardHot100::Song.number_1(song)
  #  end
  # end

end
```
and my Song Class:
```
class BillboardHot100::Song

  attr_accessor :rank, :name, :artist, :last_week, :peak_position, :weeks_on_chart, :award, :lyrics

  @@all = []

  def self.songs(song)
    self.new(
      song.css(".chart-list-item__rank").text.strip,
      song.css(".chart-list-item__title-text").text.strip,
      song.css(".chart-list-item__artist").text.strip,
      song.css(".chart-list-item__last-week").text.strip,
      song.css(".chart-list-item__weeks-at-one").text.strip,
      song.css(".chart-list-item__weeks-on-chart").text.strip,
      song.css("[class='chart-list-item__award-icon ']").text.strip,
      song.css('div.chart-list-item__lyrics a').map { |link| link['href'] }
      )
  end


  def initialize(rank=nil, name=nil, artist=nil, last_week=nil, peak_position=nil, weeks_on_chart=nil, award=nil, lyrics=nil)
    
    @rank = rank
    @name = name
    @artist = artist
    @last_week = last_week
    @peak_position = peak_position
    @weeks_on_chart = weeks_on_chart
    @award = award
    @lyrics = lyrics
    
    @@all << self
  end

  def self.all
    @@all
  end

  def self.find(index)
    self.all[index-1]
  end
  
end

```
This was working too, and was better as Objects, but I had a bunch of scraper info that really should be in my song class, so I worked further on the code until I finally got the Scraper and Song class properly separated. 

Now my Scraper class looked like this:
```
class BillboardHot100::Scraper

  def self.scrape_songs
    doc = Nokogiri::HTML(open("https://www.billboard.com/charts/hot-100"))
    doc.css("div.chart-list-item").each do |song|
      BillboardHot100::Song.new(
      rank = song.css(".chart-list-item__rank").text.strip,
      title = song.css(".chart-list-item__title-text").text.strip,
      artist = song.css(".chart-list-item__artist").text.strip,
      last_week = song.css(".chart-list-item__last-week").text.strip,
      peak_position = song.css(".chart-list-item__weeks-at-one").text.strip,
      weeks_on_chart = song.css(".chart-list-item__weeks-on-chart").text.strip,
      lyrics = song.css('div.chart-list-item__lyrics a').map { |link| link['href'] }.join,
      award = song.css('chart-list-item__award-icon').text.strip)
    end
  end

end
```

and my Song Class looked like:
```
class BillboardHot100::Song

  attr_accessor :rank, :title, :artist, :last_week, :peak_position, :weeks_on_chart, :lyrics, :award

  @@all = []

  def initialize(rank=nil, title=nil, artist=nil, last_week=nil, peak_position=nil, weeks_on_chart=nil, lyrics=nil, award=nil)
    @rank = rank
    @title = title
    @artist = artist
    @last_week = last_week
    @peak_position = peak_position
    @weeks_on_chart = weeks_on_chart
    @lyrics = lyrics
    @award = award

    @@all << self
  end

  def self.all
    @@all
  end

end

```

Allright! now we're getting somewhere! But my CLI was *not* DRY (Do Not Repeat) at all. But it was working, so I started the work of cleaning it up.

For the CLI my basic set up was this:
1. Present user with a list of ranges (1-10, 11-20, etc)
2. The user selects a range and is presented with ten songs
3. The user selects a song and is presented with more information on that song

I had #3 working well
```
def display_song(song)
    puts ""
    puts "#{song.name} by #{song.artist}"
      if song.last_week.length > 0 && !song.last_week.include?("-")
        puts "Last Week: #{song.last_week}"
      else
        puts "New To Chart!".colorize(:red)
      end
      if song.peak_position.length > 0 
        puts "Peak Position: #{song.peak_position}"
      end
      if song.weeks_on_chart.length > 0 
        puts "Weeks On Chart: #{song.weeks_on_chart}"
      end
      if song.award.length > 0 
        puts "Award: #{song.award}".colorize(:red)
      end
      if song.lyrics.length > 0 
        puts "Lyrics: #{song.lyrics.join}"
      end
  end
```

but #1 and #2 had a *lot* of repeating code.

#1
```
puts "Enter 1-10, 11-20, 21-30, 31-40, 41-50, 51-60, 61-70, 71-80, 81-90 or 91-100"
```

#2
```
def display_songs(input)
    case input.to_i
    when 1..10
      puts "Displaying songs 1 through 10"
      puts ""
      BillboardHot100::Song.all[0,10].each do |song|
        puts "#{song.rank}. #{song.name} by #{song.artist}"
      end
    when 11..20
      puts "Displaying songs 11 through 20"
      puts ""
      BillboardHot100::Song.all[10,10].each do |song|
        puts "#{song.rank}. #{song.name} by #{song.artist}"
      end

```
etc repeating for 100 songs. This is a lot of repeating code, but also, what If I wanted to reuse this code for something with 200 items? 10,000 items? 

I decide to tackle #2 first...breaking it up into two methods
```
case input
      when 1..10
        display_ten_songs(00, 10)
      when 11..20
        display_ten_songs(10, 20)
```
and using that information to display the second method
```
  def display_ten_songs(low_num, top_num)
    puts ""
    puts "Displaying songs #{low_num+1} through #{top_num}"
    puts ""
    BillboardHot100::Song.all[low_num,10].each do |song|
      puts "#{song.rank}. #{song.title} by #{song.artist}"
    end
  end
```
much cleaner! But still a lot of repeating code. Talking to a friend about some of my psuedo code about how I might make this work, he presented a novel idea to me.

```
  def quantize(input, increment)
    low_num = ((input-1)/increment).floor*increment
    display_ten_songs(low_num)
  end
```
In my case I set the increment variable to 10.

Let's work through this. Say the user inputs 86. 
86-1 = 85. 
85/increment (in this case 10) = 8.5
Using Floor we make it 8 and multiply is by the increment (10) giving us 80
And we set the low_num to 80.

Then I further abstracted my display_ten_songs method to be
```
  def display_ten_songs(low_num)
    BillboardHot100::Song.all[low_num,10].each do |song|
      puts "#{song.rank}. #{song.title} by #{song.artist}"
    end
  end
```
Much simpler!

This was the majority of my work. The remainder of my time was spent abstracting everything down. I wanted to be able to use this program with any number of objects and still work, and I believe I have. I also spent a lot of time working through cases based on how I wanted the UI to work. This was unimportant for the scope of the project, but felt good to solve so that the program behaves exactly as I want it to!

My gem: https://rubygems.org/gems/billboard_hot_100_CLI/
My github repository: https://github.com/stephenolson/billboard_hot_100

Thanks to everyone who listened to me talk about this and looked at my code!



