# Transactions table

create table public.transactions (
  id serial not null,
  user_id uuid not null,
  account_id uuid not null,
  account_mask character varying(16) null,
  statement_date date null,
  tx_date date not null,
  post_date date null,
  type character varying(10) not null,
  merchant character varying(255) null,
  description text null,
  category character varying(100) null,
  mcc integer null,
  amount numeric(12, 2) not null,
  currency character(3) null default 'USD'::bpchar,
  signed_amount numeric(12, 2) not null,
  balance numeric(14, 2) null,
  is_recurring boolean null default false,
  recurrence_id uuid null,
  original_transaction_id uuid null,
  notes text null,
  created_at timestamp with time zone null default now(),
  constraint transactions_pkey primary key (id),
  constraint transactions_user_id_fkey foreign KEY (user_id) references userlogin (id) on delete CASCADE,
  constraint transactions_type_check check (
    (
      (type)::text = any (
        (
          array[
            'debit'::character varying,
            'credit'::character varying
          ]
        )::text[]
      )
    )
  )
) TABLESPACE pg_default;

create index IF not exists idx_transactions_user_id on public.transactions using btree (user_id) TABLESPACE pg_default;

# User Login Table

CREATE  TABLE public.userlogin (
  id uuid NOT NULL,
  firstname text NULL,
  lastname text NULL,
  email text NULL,
  username text NULL,
  slug text NULL,
  avatar_url text NULL,
  timezone text NULL DEFAULT 'America/Chicago'::text,
  locale text NULL DEFAULT 'en-US'::text,
  prefs jsonb NOT NULL DEFAULT '{}'::jsonb,
  status text NOT NULL DEFAULT 'active'::text,
  created_at timestamp with time zone NOT NULL DEFAULT now(),
  updated_at timestamp with time zone NOT NULL DEFAULT now(),
  phone text NULL,
  2fa boolean NULL,
  CONSTRAINT userlogin_pkey PRIMARY KEY (id),
  CONSTRAINT userlogin_slug_key UNIQUE (slug),
  CONSTRAINT userlogin_username_key UNIQUE (username),
  CONSTRAINT userlogin_id_fkey FOREIGN KEY (id) REFERENCES auth.users(id) ON DELETE CASCADE,
  CONSTRAINT userlogin_phone_check CHECK ((length(phone) <= 10))
) TABLESPACE pg_default;
CREATE TRIGGER set_updated_at BEFORE UPDATE ON userlogin FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER sync_slug_from_username BEFORE INSERT OR UPDATE OF username ON userlogin FOR EACH ROW EXECUTE FUNCTION sync_slug_from_username();

---

# RLS policies assigned to each table

# User Login Table:

## Read transactions of logged in user

alter policy "read own userlogin"

on "public"."userlogin"

to public

using (
  (auth.uid() = id)
);

## Update own user login

alter policy "update own userlogin"

on "public"."userlogin"

to public

using (
  (auth.uid() = id)
);

# Transactions Table

## Insert own transactions

alter policy "insert own transactions"

on "public"."transactions"

to public

with check (
 (user_id = auth.uid())
);

## Read own transactions
alter policy "read own transactions"

on "public"."transactions"

to public

using (
  (user_id = auth.uid())
);

## User can delete own transactions

alter policy "Users can delete own transactions"

on "public"."transactions"

to authenticated

using (
  (user_id = auth.uid())
);

## User can edit own transactions

alter policy "Users can update own transactions"

on "public"."transactions"

to authenticated

using (
  (auth.uid() = user_id)

) with check (
  (auth.uid() = user_id)
);
