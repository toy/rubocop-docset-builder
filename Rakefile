exec 'rake' if __FILE__ == $0

require 'json'
require 'nokogiri'
require 'fspath'

FSPath.class_eval do
  def done = self / '.done'

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
  include Rake::DSL
  extend Rake::DSL

  CLEAN_DIRS = [
    ANTORA_PLAYBOOKS_DIR = 'antora-playbooks',
    ANTORA_CACHE_DIR = 'antora-cache',
    ANTORA_HTML_DIR = 'antora-html',
    HTML_DIR = 'html',
    DASHING_CONFIGS_DIR = 'dashing-configs',
    DOCSETS_DIR = 'docsets',
    ARCHIVES_DIR = 'archives',
    RESULTS_DIR = 'results',
  ]

  IGNORE_TITLES = [
    'Configurable attributes',
    'Examples',
    'References',
    'Safety',
  ]

  def self.all
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

  def self.define_tasks
    all(&:define_tasks)

    task :clean do
      rm_rf CLEAN_DIRS
    end
  end

  attr_reader :name, :url
  attr_reader :tag, :version
  attr_reader :package, :package_n_version
  attr_reader :antora_playbook_path, :dashing_config_path
  attr_reader :antora_html_path, :html_path, :docset_path, :archive_path, :result_path

  def initialize(name, url)
    @name = name
    @url = url

    @tag = fetch_latest_tag
    @version = tag.delete_prefix('v')

    @package = name.tr(' ', '_')
    @package_n_version = "#{package}-#{version}"

    @antora_playbook_path = FSPath(ANTORA_PLAYBOOKS_DIR) / "#{package_n_version}.json"
    @antora_html_path = FSPath(ANTORA_HTML_DIR) / package
    @html_path = FSPath(HTML_DIR) / package
    @dashing_config_path = FSPath(DASHING_CONFIGS_DIR) / "#{package_n_version}.json"
    @docset_path = FSPath(DOCSETS_DIR) / "#{package}.docset"
    @archive_path = FSPath(ARCHIVES_DIR) / "#{package}.docset.tgz"
    @result_path = FSPath(RESULTS_DIR) / package
  end

  def define_tasks
    file antora_playbook_path do
      antora_playbook_path.carefull_write(antora_playbook)
    end

    dir antora_html_path => antora_playbook_path do
      sh(*docker_run_args('antora/antora') + %W[
        --fetch
        --cache-dir #{ANTORA_CACHE_DIR}
        #{antora_playbook_path}
      ])
    end

    dir html_path => antora_html_path.done do
      mkdir_p html_path

      cp_r antora_html_path.glob('*'), html_path

      prepare_html
    end

    file dashing_config_path => html_path.done do
      dashing_config_path.carefull_write(dashing_config)
    end

    dir docset_path => [dashing_config_path, html_path.done] do
      mkdir_p docset_path.dirname

      sh(*%W[dashing build --source #{html_path} --config #{dashing_config_path}])

      mv docset_path.basename, docset_path.dirname

      %w[icon.png icon@2x.png].each do |icon_basename|
        ln icon_basename, docset_path
      end
    end

    file archive_path => docset_path.done do
      mkdir_p archive_path.dirname

      sh(*docker_run_args('debian') + %W[
        tar
        --directory=#{docset_path.dirname}
        --exclude=.DS_Store
        -cvzf
        #{archive_path}
        #{docset_path.basename}
      ])
    end

    docset_meta_path = result_path / 'docset.json'

    results = {
      result_path / archive_path.basename => archive_path,
      **%w[README.md icon.png icon@2x.png].to_h do |basename|
        [result_path / basename, basename]
      end,
    }

    file docset_meta_path => results.values do
      rm_rf result_path
      mkdir_p result_path

      results.each do |dst, src|
        ln src, dst
      end

      docset_meta_path.carefull_write(docset_meta)
    end

    task default: docset_meta_path
  end

private

  def dir(arg)
    fail ArgumentError, 'expected Hash with 1 pair' unless arg.is_a?(Hash) && arg.length == 1

    dir, dependencies = arg.first
    done = dir.done

    file done => dependencies do
      rm_rf dir

      yield

      touch done
    end
  end

  def fetch_latest_tag
    IO.popen(%W[git ls-remote --tags --refs --sort v:refname #{url}], &:read)
      .split("\n")
      .map{ _1.split('/').last }
      .grep(/\Av\d+(\.\d+)+\z/)
      .last
  end

  def antora_playbook
    JSON.pretty_generate({
      site: {
        title: name,
      },
      content: {
        sources: [
          {
            url:,
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
        dir: antora_html_path,
      },
    })
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

    JSON.pretty_generate({
      name:,
      package:,
      index:,
      externalURL: url,
      selectors:,
      allowJS: true, # sadly required for highlighting and alternatives are currently not easy with antora
    })
  end

  def docset_meta
    JSON.pretty_generate({
      name:,
      version:,
      archive: archive_path.basename,
      author: {
        name: 'Ivan Kuchin',
        link: 'https://github.com/toy',
      },
    })
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

RubocopDocBuilder.define_tasks
