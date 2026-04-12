exec 'rake' if __FILE__ == $0

require 'json'
require 'nokogiri'
require 'fspath'

FSPath.class_eval do
  def carefull_write(content)
    content = content&.to_s

    return if size? && read == content

    temp_file_path(dirname.mkpath) do |tmp|
      tmp.write(content)
      tmp.rename(self)
    end
  end
end

class RubocopDocBuilder
  CLEAN_DIRS = [
    PLAYBOOKS_DIR = 'playbooks',
    ANTORA_CACHE_DIR = 'antora-cache',
    HTML_DIR = 'html',
    DASHING_CONFIG_DIR = 'dashing-config',
    DOCSETS_DIR = 'docsets',
  ]

  IGNORE_TITLES = [
    'Configurable attributes',
    'Examples',
    'References',
    'Safety',
  ]

  def self.all
    return to_enum(__method__) unless block_given?

    yield new 'RuboCop',              'https://github.com/rubocop/rubocop.git'
    yield new 'RuboCop Capybara',     'https://github.com/rubocop/rubocop-capybara.git'
    yield new 'RuboCop factory_bot',  'https://github.com/rubocop/rubocop-factory_bot.git'
    yield new 'RuboCop Minitest',     'https://github.com/rubocop/rubocop-minitest.git'
    yield new 'RuboCop Performance',  'https://github.com/rubocop/rubocop-performance.git'
    yield new 'RuboCop Rails',        'https://github.com/rubocop/rubocop-rails.git'
    yield new 'RuboCop RSpec',        'https://github.com/rubocop/rubocop-rspec.git'
    yield new 'RuboCop RSpec Rails',  'https://github.com/rubocop/rubocop-rspec_rails.git'
    yield new 'RuboCop Packaging',    'https://github.com/utkarsh2102/rubocop-packaging.git'
    yield new 'RuboCop ThreadSafety', 'https://github.com/rubocop/rubocop-thread_safety.git'
  end

  attr_reader :name, :url
  attr_reader :name, :package, :name_n_version
  attr_reader :playbook_path, :html_path, :dashing_config_path, :docset_path, :archive_path

  def initialize(name, url)
    @name = name
    @url = url

    @package = name.tr(' ', '_')
    @name_n_version = "#{name}-#{version}"

    @playbook_path = FSPath(PLAYBOOKS_DIR) / "#{name_n_version}.json"
    @html_path = FSPath(HTML_DIR) / name
    @dashing_config_path = FSPath(DASHING_CONFIG_DIR) / "#{name_n_version}.json"
    @docset_path = FSPath("#{package}.docset")
    @archive_path = FSPath(DOCSETS_DIR) / package / "#{package}.docset.tgz"
  end

  def tag
    @tag ||=
      IO.popen(%W[git ls-remote --tags --refs --sort v:refname #{url}], &:read)
        .split("\n")
        .map{ _1.split('/').last }
        .grep(/\Av\d+(\.\d+)+\z/)
        .last
  end

  def version
    @version ||= tag.delete_prefix('v')
  end

  def build
    run_antora

    prepare_html

    run_dashing

    create_archive

    fill_docsets_dir
  end

private

  def playbook
    object = {
      site: {
        title: name,
        url: "https://docs.rubocop.org/#{File.basename(url, '.git')}",
      },
      content: {
        sources: [
          {
            url: url,
            branches: nil,
            tags: tag,
            start_path: 'docs',
          },
        ],
      },
      asciidoc: {
        attributes: {
          experimental: '',
          idprefix: '',
          idseparator: '-',
          linkattrs: '',
          toc: nil,
          'page-pagination': '',
        },
      },
      ui: {
        bundle: {
          url: 'https://gitlab.com/antora/antora-ui-default/-/jobs/artifacts/master/raw/build/ui-bundle.zip?job=bundle-stable',
          snapshot: true,
        },
        supplemental_files: 'supplemental-ui',
      },
      output: {
        dir: html_path,
      },
    }

    JSON.pretty_generate(object)
  end

  def dashing_config
    index = html_path.glob('{,*/}*/index.html').first
    abort "didn't find index for #{name} at #{html_path}" unless index

    selectors = {}

    %w[
      category
      guide
      section
      setting
      test
    ].each do |type|
      selectors["[data-type=#{type}]:not([title])"] = type.capitalize
      selectors["[data-type=#{type}][title]"] = {attr: 'title', type: type.capitalize}
    end

    config = {
      name: name,
      package: package,
      index: index,
      externalURL: url,
      selectors: selectors,
      allowJS: true, # sadly required for highlighting and alternatives are currently not easy with antora
    }

    JSON.pretty_generate(config)
  end

  def docset_meta
    meta = {
      name: name,
      version: version,
      archive: archive_path.basename,
      author: {
        name: 'Ivan Kuchin',
        link: 'https://github.com/toy',
      },
    }

    JSON.pretty_generate(meta)
  end

  def run_antora
    playbook_path.carefull_write(playbook)

    html_path.rmtree

    abort unless system(*docker_run_args('antora/antora') + %W[
      --fetch
      --cache-dir #{ANTORA_CACHE_DIR}
      #{playbook_path}
    ])

    (html_path / '404.html').unlink
  end

  def prepare_html
    html_path.glob('**/*.html') do |html_path|
      doc = Nokogiri::HTML(html_path.read)

      doc.search('header, aside, .nav-container, .toolbar, nav').remove

      cops = html_path.basename.to_s.start_with?('cops_')

      index = []
      doc.search('h1, h2, h3, h4').each do |h|
        title = h.text.strip
        level = h.name[1].to_i
        index[level - 1..] = title
        next if IGNORE_TITLES.include?(title)

        type = if cops
          case level
          when 1
            'category'
          when 2
            title.include?('/') ? 'test' : 'setting'
          when 3
            h['title'] = index.drop(1).join(': ')
            'section'
          when 4
            h['title'] = index.values_at(1, 3).join(': ')
            title.include?(':') ? 'setting' : 'section'
          end
        else
          if level == 1
            'guide'
          else
            'section'
          end
        end

        h['data-type'] = type if type
      end

      html_path.carefull_write(doc)
    end
  end

  def run_dashing
    dashing_config_path.carefull_write(dashing_config)

    docset_path.rmtree

    abort unless system(*%W[dashing build --source #{html_path} --config #{dashing_config_path}])

    %w[icon.png icon@2x.png].each do |icon_basename|
      (docset_path / icon_basename).make_link(icon_basename)
    end
  end

  def create_archive
    abort unless system(*docker_run_args('debian') + %W[
      tar
      --exclude=.DS_Store
      -cvzf
      #{archive_path.basename}
      #{docset_path}
    ])
  end

  def fill_docsets_dir
    archive_dir = archive_path.dirname

    archive_dir.rmtree
    archive_dir.mkpath

    archive_path.make_link(archive_path.basename)

    %w[README.md icon.png icon@2x.png].each do |basename|
      (archive_dir / basename).make_link(basename)
    end

    (archive_dir / 'docset.json').carefull_write(docset_meta)
  end

  def docker_run_args(image)
    %W[
      docker run
      --user #{Process.uid}:#{Process.gid}
      --rm
      -i
      -v #{Dir.pwd}:/here
      -w /here
      #{image}
    ]
  end
end

task :default do
  RubocopDocBuilder.all(&:build)
end

task :clean do
  rm_rf RubocopDocBuilder::CLEAN_DIRS + Dir[*%w[*.docset *.docset.tgz]]
end
