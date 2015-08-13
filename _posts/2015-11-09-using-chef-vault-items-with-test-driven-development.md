---
layout:     post
title:      Using Chef Vault with TDD
date:       2015-11-09
summary:    Automate encryption of your ssh keys on server using TDD
---

Chef-vault is a way to store your ssh keys in the chef server. Chef vault uses a data bag to encrypt the keys and decrypts them when requested.

Suppose, we have a json file, <strong>secrets.json</strong> containing ssh keys pairs, for example
<pre>{
  "key_1_name": "secret_1",
  "key_2_name": "secret_2"
}
</pre>
The first step would be encrypt these keys and store in the chef-vault. The keys will be stored in a databag within a dataset. Let us call our databag <strong>secrets</strong> and the dataset as <strong>application_secrets</strong>. An easy way to do create the databag at command line is,

<pre>
bundle exec knife vault create secrets application_secrets --json ./secrets.json
</pre>

This stores the secret keys in our chef-vault.

The next step would be to retrieve them in our recipe. We will use help of a community cookbook <strong>chef-vault</strong> to access chef-vault items.

Let us start by writing an integration test.

#### Integration testing
This integration test makes sure that there is a file on our chef-node when chef client runs on it. The chef-client run fetches the secrets keys from the chef server and dumps them in a file on the node. We will use <strong>kitchen</strong> and <strong>serverspec</strong> to do it.

#### Steps

* Mock the databags for the kitchen vagrant machine. Let us specify the path for the mock databag item in the <strong>kitchen.yml</strong> file like this to be,
<pre>
suites:
  - name: secrets_integration_test
    run_list: [ 'recipe[secrets::default]' ]
    attributes: {
      'dev_mode': true,
      'data_bags_path': 'test/integration/secrets_integration_test/data_bags',
  }
</pre>
* Make the mock databag file, <strong>test/integration/secrets_integration_test/data_bags/secrets/application_secrets.json</strong> like this,
<pre>
{
  "test_key_1": "test_value_1",
  "test_key_2": "test_value_2"
}
</pre>

* The path of the databag file is comprised of the databag name <strong>secrets</strong> and the dataset name <strong>application_secrets</strong>.

The integration test will look something like this,
<pre>
describe file('/application_secrets.json') do
  its(:content) { should match(/test_value_1/) }
  its(:content) { should match(/test_value_2/) }
end
</pre>

#### Unit testing

Using chefspec, a unit test can be written like,
<pre>describe 'secrets::default' do
  let(:chef_run) { ChefSpec::Runner.new }

  before do
    allow(ChefVault::Item).to receive(:load).
      with('secrets', 'application_secrets').and_return({
        'test_key_1' =&gt; 'test_value_1',
        'test_key_2' =&gt; 'test_value_2'} )

    chef_run.converge(described_recipe)
  end

  it 'creates the application_secrets file with the secret key values in it' do
    expect(chef_run).to render_file('/application_secrets.json')
      .with_content(/test_value_1/)

    expect(chef_run).to render_file('/application_secrets.json')
      .with_content(/test_value_2/)
  end
end
</pre>
#### Accessing secrets in the recipe

With the help of the chef_vault recipe, the secret keys can be easily accessed,
<pre>
  my_secrets = chef_vault_item('secrets','application_secrets')
</pre>

<strong>my_secrets</strong> is a ruby hash that can be used in the recipe as required.

A file can be rendered using this hash like this,
<pre>
require 'json'

my_secrets = chef_vault_item('secrets','application_secrets')

file '/application_secrets.json' do
  content my_secrets.to_json
  mode '0640'
  group 'root'
  sensitive true
end
</pre>
* The block configuration `sensitive true` hides the content of the file, i.e the key values during a chef client run on STDOUT or in the logs. The content is just showed as hidden sensitive content
