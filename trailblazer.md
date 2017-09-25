# Why do we need trailblazer?

> more intuitive app structure and easier to navigate

> embraces the component structure

> simplifies refactoring

### Standart tasks and solution with traiblazer and pure rails application

### Case 1
##### GOAL: Remove all business logic from the model and provide a separate, streamlined object for it.

Callbacks in models

```ruby
class Company < ActiveRecord::Base
  # ... other code omitted
  
  after_create :send_welcome_email!
  
  private

  def send_welcome_email!
    # send email logic
  end
end

```

> Why it's bad?
> Someone want to create entity through rails console -> need to skip callback or remove email sending from dev/test env
> We hardcoded this logic

```ruby
  class Company::Create < Trailblazer::Operation
    # ... other code omitted
    
    step :send_welcome_email!
  
    def send_welcome_email!(options, *)
      # send email logic
    end
  end
```

> Why it's good?
> We delegated part of creation logic to operation, so our model isn't hardcoded
> of course we can easily run through rails console `Company::Create.({name: 'test'})`

### Case 2
##### GOAL: Remove all business logic from the model and provide a separate, streamlined object for it.

Validation in model

We want to add ability to create Company without info, but later we want
to validate these fields when someone want too update this company

```ruby
  class Company < ActiveRecord::Base
    validates :name,             absence:  true, on: :create
    validates :short_description, absence:  true, on: :create
    validates :full_description,  absence:  true, on: :create
    
    validates :name,             presence: true, length: { in: 6..20 }, on: :update
    validates :short_description, presence: true, length: { in: 6..20 }, on: :update
    validates :full_description,  presence: true, length: { in: 20..50 }, on: :update
    
    # ... other code omitted
  end
```

As we can see it's easy to do it with pure rails application. 
But what about delegation the logic to other entities (create and 
update validation logic)?
And what about case when when we have 
super big model with multiple callbacks, associations etc... Do you think you can easily understand 
what's going on (especially with nested entities updating....)

```ruby
# app/concepts/company/contract/create.rb
--------------------------------------------------
module Company::Contract
  class Create < Reform::Form
    property :name
    property :short_description
    property :full_description
  end
end
```

```ruby
# app/concepts/company/contract/create.rb
--------------------------------------------------
module Company::Contract
  class Update < Reform::Form
    property :name
    property :short_description
    property :full_description

    validates :name, length: 6..20
    validates :short_description, length: 6..20
    validates :full_description, length: 20..50
  end
end
```

> Why it's good?
> Now it looks cleaner -> we separate validation for creating/updating

### Case 3

##### GOAL: Delegate authorization to separate policy class

Authorization examples

```ruby
class Company::Create < Trailblazer::Operation
  step Model( Company, :new )
  step Policy::Pundit(CompanyPolicy, :create?)
  
  # ... other code omitted
end

```

```ruby
class CompanyPolicy < ApplicationPolicy
  def create?
    # pass logic
  end
end
```

> Why it's good?
> It's super clear for developers -> in operation we have step for authorization.
> In pundit policies, it is a convention to have access to those objects at runtime 
> and build rules on top of those.

### Case 4

##### GOAL: Remove all business logic from the controller and provide a separate, streamlined object for it.

Business logic in controllers. They end up as lean HTTP endpoints.
So no business logic is to be found in the controller. 

What Trailblazer provides: OPERATION

```ruby
class CompaniesController < ApplicationController
  # ... other code omitted
  
  def create
    @company = Company.new(company_params)
    @company.created_by = current_user
    if @company.save
      render json: @company
    else
      render json: @company.errors, status: :unprocessable_entity
    end
  end
  
  ###

  private
  def company_params
    params.require(:company).permit(:name)
  end
end

```

> Why it's bad?
> We keep part of logic in controller's methods -> BUT IT'S JUST ENDPOINT FOR APP!!

```ruby
class CompaniesController < ApplicationController
  # ... other code omitted
  
  def create
    res = run Company::Create do |op|
      return render json: op.model
    end
    render json: { errors: res.errors }, status: :unprocessable_entity
  end
  
  ###
end

```

> Why it's good?
> We keep controllers super clean and simply run needed operation

### Case 5

##### GOAL: Add ability to easily test application

Testing

```ruby
# app/controllers/companies_controller.rb

class CompaniesController < ApplicationController
  # ... other code omitted

  def new
    @company = Company.new
    %w(home office mobile).each do |phone|
      @company.phones.build(phone_type: phone)
    end
  end
end
```

```ruby
# spec/controllers/contacts_controller_spec.rb

# ... other specs omitted

describe 'GET #new' do
  it 'assigns a home, office, and mobile phone to the new company' do
    get :new
    assigns(:company).phones.map{ |p| p.phone_type }.should eq %w(home office mobile)
  end
end
```

> Why it's bad?
> It's strange to check controller variables
> It's hardcoded in controller and don't reusable

```ruby
class CompaniesController < ApplicationController
  # ... other code omitted

  def new
    form Company::Create
  end
end
```

```ruby
class Company < ActiveRecord::Base
  class Create < Reform::Form
    property :name
    collection :phones, populate_if_empty: :populate_phones!
    
    validate :name, presence: true
  end
  
  private
  def populate_phones!(fragment, **)
    Phone.build(phone_type: fragment['name'])
  end
end  
```


So now we can test entity creation and don't test it through controllers
```ruby
describe Company::Create do
  it 'prohibits empty params' do
    result = Company::Create.({})

    expect(result).to be_failure
    expect(result['model']).to be_new
  end
end
```

> Why it's good?
> Since operations embrace the entire workflow for an application’s function, 
> you can write simple and fast unit-tests to assert the correct behavior.

### Case 6

##### GOAL: Provide an object that can render a template.

Business logic in views.
In pure Rails app we can't easily encapsulate code in view layer. 

What Trailblazer provides: CELL (an object that can render a template)
In fact Cells are faster than ActionView.

```ruby
  # app/views/companies/show.haml
  
  %span
    Created By #{creator_title(@company)}

```

```ruby
  # app/helpers/company_helper.rb
  
  module CompanyHelper
    def creator_title(company)
      company.creator.is_a?(Admin) ? 'Superman' : 'Regular guy'
    end
  end

```

> Why it's bad?
> We keep logic in helpers -> it's not object-oriented


```ruby
  # app/concepts/companies/views/show.haml
  
  %span
    "Created By #{creator_title}"

```

```ruby
  class Company::Cell < Cell::ViewModel
    def show
      render
    end
    
    private
    def creator_title
      model.creator.is_a?(Admin) ? 'Superman' : 'Regular guy'
    end
  end
```
