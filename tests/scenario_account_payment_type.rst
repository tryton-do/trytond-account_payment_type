=============================
Invoice Payment Type Scenario
=============================

Imports::

    >>> import datetime
    >>> from dateutil.relativedelta import relativedelta
    >>> from decimal import Decimal
    >>> from operator import attrgetter
    >>> from proteus import config, Model, Wizard
    >>> from trytond.modules.company.tests.tools import create_company, \
    ...     get_company
    >>> from trytond.modules.account.tests.tools import create_fiscalyear, \
    ...     create_chart, get_accounts, create_tax, set_tax_code
    >>> from.trytond.modules.account_invoice.tests.tools import \
    ...     set_fiscalyear_invoice_sequences, create_payment_term
    >>> today = datetime.date.today()

Create database::

    >>> config = config.set_trytond()
    >>> config.pool.test = True

Install account_invoice::

    >>> Module = Model.get('ir.module')
    >>> account_invoice_module, = Module.find(
    ...     [('name', '=', 'account_payment_type')])
    >>> Module.install([account_invoice_module.id], config.context)
    >>> Wizard('ir.module.install_upgrade').execute('upgrade')

Create company::

    >>> _ = create_company()
    >>> company = get_company()

Reload the context::

    >>> User = Model.get('res.user')
    >>> config._context = User.get_preferences(True, config.context)

Create fiscal year::

    >>> fiscalyear = set_fiscalyear_invoice_sequences(
    ...     create_fiscalyear(company))
    >>> fiscalyear.click('create_period')
    >>> period = fiscalyear.periods[0]

Create chart of accounts::

    >>> _ = create_chart(company)
    >>> accounts = get_accounts(company)
    >>> receivable = accounts['receivable']
    >>> revenue = accounts['revenue']
    >>> expense = accounts['expense']
    >>> cash = accounts['cash']

Create tax::

    >>> tax = set_tax_code(create_tax(Decimal('.10')))
    >>> tax.save()

Create product::

    >>> ProductUom = Model.get('product.uom')
    >>> unit, = ProductUom.find([('name', '=', 'Unit')])
    >>> ProductTemplate = Model.get('product.template')
    >>> Product = Model.get('product.product')
    >>> product = Product()
    >>> template = ProductTemplate()
    >>> template.name = 'product'
    >>> template.default_uom = unit
    >>> template.type = 'service'
    >>> template.list_price = Decimal('50')
    >>> template.cost_price = Decimal('25')
    >>> template.account_expense = expense
    >>> template.account_revenue = revenue
    >>> template.customer_taxes.append(tax)
    >>> template.save()
    >>> product.template = template
    >>> product.save()

Create payment term::

    >>> payment_term = create_payment_term()
    >>> payment_term.save()

Create payment type::

    >>> PaymentType = Model.get('account.payment.type')
    >>> receivable = PaymentType(name='Receivable', kind='receivable')
    >>> receivable.save()
    >>> payable = PaymentType(name='Payable', kind='payable')
    >>> payable.save()

Create party::

    >>> Party = Model.get('party.party')
    >>> party = Party(name='Party')
    >>> party.customer_payment_type = receivable
    >>> party.supplier_payment_type = payable
    >>> party.save()

Check invoice payment type is correctly assigned::

    >>> Invoice = Model.get('account.invoice')
    >>> invoice = Invoice()
    >>> invoice.party = party
    >>> invoice.payment_term = payment_term
    >>> invoice.payment_type
    >>> line = invoice.lines.new()
    >>> line.product = product
    >>> line.quantity = 1
    >>> line.unit_price = Decimal('50.0')
    >>> invoice.untaxed_amount
    Decimal('50.00')
    >>> invoice.payment_type == receivable
    True
    >>> line = invoice.lines.new()
    >>> line.product = product
    >>> line.quantity = -1
    >>> line.unit_price = Decimal('40.0')
    >>> invoice.payment_type == receivable
    True

When its a return its ussed the supplier payment_kind::

    >>> line = invoice.lines.new()
    >>> line.product = product
    >>> line.quantity = -1
    >>> line.unit_price = Decimal('40.0')
    >>> invoice.untaxed_amount
    Decimal('-30.00')
    >>> invoice.payment_type == payable
    True

And where clearing all the lines the recevaible payment type is used::

    >>> _ = invoice.lines.pop()
    >>> _ = invoice.lines.pop()
    >>> _ = invoice.lines.pop()
    >>> invoice.payment_type == receivable
    True
    >>> invoice.untaxed_amount
    Decimal('0.00')