require "rake/clean"
require "oga"
require "zlib"
require "rubygems/package"
require "epub/maker/task"

EPUB::Parser::XMLDocument.backend = :Oga

require "open-uri"
class DownloadTask < Rake::TaskLib
  def initialize(uri)
    if uri.kind_of?(Hash)
      @to = uri.keys.first
      @uri = URI(uri.values.first)
    else
      @uri = URI(uri)
      @to = @uri.path.pathmap("%f")
    end
    define
  end

  private

  def define
    desc "Download from #{@uri}"
    file @to do |t|
      File.write t.name, @uri.read
    end
  end
end

def download(args)
  DownloadTask.new args
end

# TODO: Modify CSS to style index.xhtml
def load_html(file_path)
  content = File.read(file_path, mode: "rb", encoding: "ISO-2022-JP").encode("UTF-8", invalide: :replace, undef: :replace)

  case file_path.pathmap("%n")
  when "preface"
    content.sub! /<p class="right"/, '<p class="right">'
  when "gc"
    content.sub! %r|しかも参照しているオブジェクト同士が近くに来るので</li>|, "しかも参照しているオブジェクト同士が近くに来るので<br>"
  when "iterator"
    content.sub! %r|<code>pcall</code>がある。</li>|, "<code>pcall</code>がある。<br>"
    content.sub! /この引数は引数チェックの厳しさを変えるだけなのでどうでもいい。/, "この引数は引数チェックの厳しさを変えるだけなのでどうでもいい。<br>"
  end

  doc = Oga.parse_html(content)

  case file_path.pathmap("%n")
  when "yacc"
    title = doc.xpath("//title").first
    title.inner_text = title.text
  end

  doc.doctype = nil
  html = doc.xpath("/html").first
  [
    [nil, "xmlns", "xhtml"],
    ["xmlns", "epub", "epub"]
  ].each do |(prefix, name, key)|
    html.add_attribute Oga::XML::Attribute.new(namespace_name: prefix, name: name, value: EPUB::NAMESPACES[key])
  end
  doc.css('meta[http-equiv="Content-Type"]').first.set("content", "text/html; charset=utf-8")
  doc.css('meta[http-equiv="Content-Language"]').first.remove
  made = doc.css('link[rev]').first
  made.unset "rev"
  made.set "rel", "author"
  doc
end

SRC_URI = URI("http://i.loveruby.net/ja/rhg/ar/RubyHackingGuide.tar.gz")
SRC = "src"
BUILD = "build"
DEST = SRC_URI.to_s.pathmap("%n").ext(".epub")

desc "Run epubcheck"
task :epubcheck => DEST do |t|
  sh t.name, t.source
end

desc "Build EPUB file"
file DEST => :epub_tree do |t|
  EPUB::Maker.archive BUILD, t.name
end
CLEAN.include DEST

directory "#{BUILD}/META-INF"

directory "#{BUILD}/OPS"

file "#{BUILD}/OPS/index.xhtml" => "#{SRC}/index.html" do |t|
  doc = load_html(t.source)
  body = Oga::XML::Element.new(name: "body")
  nav = Oga::XML::Element.new(name: "nav", attributes: [Oga::XML::Attribute.new(namespace_name: "epub", name: "type", value: "toc")])
  stack = [nav]
  processing_toc = false
  current_body = doc.xpath("/html/body").first
  current_body.children.each do |node|
    unless node.kind_of? Oga::XML::Element
      body.children << node
      next
    end
    if node.text == "目次"
      processing_toc = true
      body.children << nav
      nav.children << node
    end
    if node.text == "ライセンスなど"
      processing_toc = false
    end
    if processing_toc
      case node.name
      when "h2"
        stack.last.children << node
        ol = Oga::XML::Element.new(name: "ol")
        stack.last.children << ol
        stack << ol
      when "h3"
        stack.push node
      when "ul"
        li = Oga::XML::Element.new(name: "li")
        if stack.last.name == "h3"
          h3 = stack.pop
          h3.name = "span"
          li.children << h3
        else
          # FIXME
          span = Oga::XML::Element.new(name: "span")
          span.inner_text = "()"
          li.children << span
        end
        node.name = "ol"
        li.children << node
        stack.last.children << li
      end
    else
      body.children << node
    end
  end
  current_body.remove
  doc.xpath("/html").first.children << body
  doc.xpath("//a").each do |a|
    a["href"] = a["href"].sub(/\.html\z/, ".xhtml")
  end
  doc.instance_variable_set :@type, "xml" # FIXME
  File.write t.name, doc.to_xml
end

rule %r|^#{BUILD}/OPS/.+\.xhtml| => "%{^#{BUILD}/OPS,#{SRC}}X.html" do |t|
  doc = load_html(t.source)
  doc.instance_variable_set :@type, "xml" # FIXME
  File.write t.name, doc.to_xml
end
rule %r|^#{BUILD}/OPS/.+\.css| => "%{^#{BUILD}/OPS,#{SRC}}X.css" do |t|
  content = File.read(t.source, mode: "rb", encoding: "ISO-2022-JP").encode("UTF-8", invalide: :replace, undef: :replace)
  File.write t.name, content
end
rule %r|^#{BUILD}/OPS/.+\.jpg| => "%{^#{BUILD}/OPS,#{SRC}}X.jpg" do |t|
  copy t.source, t.name
end
directory BUILD
CLEAN.include BUILD

file "rakelib/build.rake" => ["RubyHackingGuide.tar.gz", "rakelib"] do |t|
  Gem::Package::TarReader.new(Zlib::GzipReader.new(File.open(t.source))).each do |entry|
    next unless entry.file?
    base = entry.full_name.pathmap("%1d")
    path = entry.full_name.pathmap("%{^#{base},#{SRC}}p")
    dir = path.pathmap("%d")
    mkpath(dir) unless File.exist?(dir)
    File.write path, entry.read
  end

  rakefile = <<~RAKEFILE
    require "epub/maker"

    EPUB_FILES = FileList["#{SRC}/**/*"].pathmap("%{^#{SRC},#{BUILD}/OPS}p").pathmap("%{\.html$,.xhtml}p")

    desc "Build directory tree for building EPUB content files"
    multitask epub_tree: EPUB_FILES + ["#{BUILD}/META-INF/container.xml", "#{BUILD}/package.opf"]

    file "#{BUILD}/META-INF/container.xml" => ["#{BUILD}/package.opf", "#{BUILD}/META-INF"] do |t|
      container = EPUB::OCF::Container.new
      container.make_rootfile full_path: t.source.pathmap("%{^#{BUILD}/,}p")

      File.write t.name, container.to_xml
    end

    file "#{BUILD}/package.opf" => EPUB_FILES do |t|
      nav_file = "#{BUILD}/OPS/index.xhtml"
      index = Oga.parse_xml(open(nav_file))
      title = index.xpath("//title").first.text
      creator = index.css('a[href^="mailto"]').first.text.split(/\s+/).first

      package = EPUB::Publication::Package.new

      package.make_metadata do |metadata|
        metadata.title = title
        metadata.language = "ja"
        metadata.creator = creator
        metadata.modified = "2004-07-20T23:08:12Z"
      end

      manifest = package.make_manifest {|manifest|
        t.sources.each do |file|
          next unless File.file? file
          href = file.pathmap("%{^#{BUILD}/,}p")
          item_options = {
            id: href.gsub(/[\\\/.]/, "-"),
            href: href
          }
          if file == nav_file
            item_options[:properties] = ["nav"]
          end
          manifest.make_item item_options
        end
      }

      package.make_spine do |spine|
        spine.make_itemref do |ir|
          ir.item = manifest.nav
          ir.linear = true
        end
        nav = EPUB::ContentDocument::Navigation.new
        nav.navigations = EPUB::Parser::ContentDocument.new(manifest.nav).parse_navigations(index)
        nav.toc.traverse do |item, _|
          if item.item
            spine.make_itemref do |ir|
              ir.item = item.item
              ir.linear = true
            end
          end
        end
      end

      File.write t.name, package.to_xml
    end

    EPUB_FILES.each do |path|
      src = path.pathmap("%{^#{BUILD}/OPS,#{SRC}}p").pathmap("%{\.xhtml,.html}p")
      dir = path.pathmap("%d")
      if File.file? src
        directory dir
        file path => dir
      end
    end
  RAKEFILE

  File.write t.name, rakefile
end
directory "rakelib"

import "rakelib/build.rake"
CLEAN.include "rakelib/build.rake"

directory SRC
CLOBBER.include SRC

download "RubyHackingGuide.tar.gz" => SRC_URI
CLOBBER.include "RubyHackingGuide.tar.gz"
