---
layout: post
title: "Double entry Bookkeeping is a Requirement"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Good accounting is easy, but unpopular

Nearly every company/startup that I've worked at, there has been some kind of basic accounting system
implemented, even if the company never deals with money directly.  However, I rarely see people talk
about implementing accounting systems probably because it's not easily understood by hackers, and it's
not something "fun" to talk about, but good accounting will actually help the core product.  

Economics is used everywhere, even when printed money is not the form of currency.  For example, you
run a dropbox competitor and you'd like to offer more storage if the user invites more friends.  You're
exchanging user acquisition for storage space.  If you were to implement this as a green developer,
you'd probably have some field in the user model that indicates the max storage space allotted to the
user, and you'd increment this number every time the user invites a friend.

There's probably nothing inherently wrong with doing it this way, it's a simple approach, but leaves
the door open for abuse should someone find a way to increment the field without inviting any friends.

### Double-entry is built-in error correction

The idea behind [double entry bookkeeping](http://en.wikipedia.org/wiki/Double-entry_bookkeeping_system) 
is that value is never created or destroyed.  "Recording of a debit amount to one account and an equal credit amount to another account results in total debits being equal to total credits for all accounts in the general ledger. If the accounting entries are recorded without error, the aggregate balance of all accounts having positive balances will be equal to the aggregate balance of all accounts having negative balances"

I'll give a brief example of how one should implement this in their system.  My snippets of code here 
can be found in the [example-ledger project on github](http://github.com/deanmao/example-ledger).  
Developers should be able to retrofit my example code in their own projects in 30 minutes or less.  I'll
be using Rails to demonstrate the idea, but one could easily extract the concepts into a django or 
flask project.

### A simple implementation

First you'll need an account model with a few fields.  In the example below, user_id isn't necessary, 
but in most applications, users would be associated with an account.  The account could be used to 
represent how much "virtual currency" they have in the bank, or how much dropbox storage space they've
earned through inviting friends.

    create_table :accounts do |t|
      t.string :name
      t.integer :user_id
      t.string :type
    end
    
The name can be anything you like, and the type field will be used for adding subclasses of account.
An example subclass might be Asset, Expense, or Sales.

    class Account < ActiveRecord::Base
      attr_accessible :name
  
      belongs_to :user
      has_many :entries

      validates :name, :presence => true

      class << self
        def accounts_receivable
          Asset.find_or_create_by_name('accounts receivable')
        end

        def accounts_payable
          Liability.find_or_create_by_name('accounts payable')
        end

        def sales
          Income.find_or_create_by_name('sales')
        end

        def miscellaneous_expense
          Expense.find_or_create_by_name('misc expense')
        end
      end

      def side
        nil
      end

      def balance(start_date = 10.years.ago, end_date = (Date.today + 1.day))
        self.entries.joins(:ftransaction)
              .where(["ftransactions.state = ?", 'memo']).sum(:amount) || 0
      end
    end

Here's an example subclass:

    class Asset < Account
      def side
        :debit
      end
    end

The idea here is simple -- we have an account for any place we want our money to go, and we label these accounts
as an asset, liability, or expense.  These are based on the standard accounting model where [Assets = Liability + Owner's Equity](http://en.wikipedia.org/wiki/Accounting_equation)
Income (sales) and Liability are credit balancing, and Asset and Expense are debit balancing.  You won't have to think
too much about this when you're writing the accounting system, you can simply take this as a given for now.

We implement debit accounts to have a normal positive balance, and credit accounts to have a normal negative balance.  When
you sum up the amounts from all accounts, it should add up to zero.  

### The Entry class

Next, we'll need a model that stores the actual dollar amount and tells us which account it belongs to.  We'll structure this
as an Entry model.  The Ftransaction that you see everywhere is simply the financial transaction which is a list of entries
whose sum should add to zero.  In the accounting world, this would be a single point where money was moved around.

#### The schema:

    create_table "entries" do |t|
      t.integer  "amount"
      t.integer  "account_id"
      t.integer  "ftransaction_id"
      t.boolean  "source"
      t.string   "type"
    end

#### The code:

    class Entry < ActiveRecord::Base
      attr_accessible :account_id, :amount, :ftransaction_id, :source, :type

      belongs_to :ftransaction
      belongs_to :account

      def side_amount
        if amount && :credit == account.side
          return -amount
        else
          return amount || 0
        end
      end

      def side_amount=(amt)
        if :credit == account.side
          self.amount = -amt
        else
          self.amount = amt
        end
      end
    end

### The Transaction class

This is the last class needed to appropriately represent money moving around.  It represents a single action where money is
moved -- for example if you went to the store and bought a pack of gum, according to the store's ledger, it might be
represented like this:

income: $2.00

receivables: $1.80

sales tax: $0.20

Income is credit, receivables and sales tax expense are debits.  Receivables are what you actually might get in the bank
account, and the sales tax records what you'll probably have to pay the government later on.  If you're developing a web
store, you'll probably have to do something similar.  If you're developing something that might involve virtual currency,
you may have something similar, but with different account representations.

#### The schema:

    create_table "ftransactions", :force => true do |t|
      t.string   "state"
      t.string   "type"
    end

#### The code:

    class Ftransaction < ActiveRecord::Base
      attr_accessible :state, :type

      has_many :entries, :dependent => :destroy

      # The following entries are helpful, but not necessary
      has_one :sales_entry, :class_name => 'SalesEntry'
      has_one :source_entry, :class_name => 'Entry', :conditions => {:source => true}

      validate :check_zero_entries
      validate :check_unbalanced_entries

      state_machine :state, :initial => :memo do
        event :post do
          transition :memo => :posted
        end
      end

      def balanced?
        self.entries.inject(0){|sum,entry| sum + (entry.amount || 0)} == 0
      end

      private
      def check_zero_entries
        self.errors[:base] << "there are no entries" unless self.entries.size > 0
      end

      def check_unbalanced_entries
        self.errors[:base] << "entries are unbalanced" unless balanced?
      end
    end





