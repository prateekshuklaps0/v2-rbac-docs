// V2 Multi-Org Lead Management Schema
// Paste into https://dbdiagram.io/d

// ============================================
// CORE ENTITIES
// ============================================

Table organizations {
  id integer [pk, increment]
  name varchar [not null]
  internal_id varchar [not null, unique]
  features text
  limitations jsonb
  features_override jsonb
  contact_email varchar
  contact_phone varchar
  logo varchar
  registered_at timestamp
  registered_by integer [ref: > users.id]
  created_at timestamp
  updated_at timestamp
}

Table users {
  id integer [pk, increment]
  email varchar [not null, unique]
  password_hash varchar [not null]
  role varchar [note: 'admin, super_admin, student, counsellor, user']
  sub_role varchar
  super_admin_features text
  status varchar [note: 'active, disabled, invited']
  name varchar
  phone varchar
  country_code varchar
  image varchar
  school_id integer [ref: > schools.id]
  organization_id integer [ref: > organizations.id]
  last_logged_in_at timestamp
  login_count integer
  created_at timestamp
  updated_at timestamp
}

Table schools {
  id integer [pk, increment]
  org_id integer [not null, ref: > organizations.id]
  name varchar [not null]
  address jsonb
  contact_email varchar
  contact_phone varchar
  description text
  is_active boolean [default: true]
  created_by integer [ref: > users.id]
  created_at timestamp
  updated_at timestamp
}

Table org_users {
  id integer [pk, increment]
  org_id integer [not null, ref: > organizations.id]
  user_id integer [not null, ref: > users.id]
  org_role varchar
  features text
  status varchar
  display_name varchar
  display_email varchar
  is_primary boolean
  created_at timestamp
  updated_at timestamp

  indexes {
    (org_id, user_id) [unique]
  }
}

Table counsellors {
  id integer [pk, increment]
  user_id integer [ref: > users.id]
  org_id integer [ref: > organizations.id]
  name varchar
  email varchar
  phone varchar
  is_active boolean
  created_at timestamp
  updated_at timestamp
}

// ============================================
// PROGRAMS & PIPELINE
// ============================================

Table programs {
  id integer [pk, increment]
  org_id integer [not null, ref: > organizations.id]
  name varchar [not null]
  description text
  code varchar
  is_active boolean [default: true]
  school_id integer [ref: > schools.id]
  metadata jsonb
  created_at timestamp
  updated_at timestamp
}

Table batches {
  id integer [pk, increment]
  org_id integer [not null, ref: > organizations.id]
  school_id integer [ref: > schools.id]
  program_id integer [ref: > programs.id]
  name varchar [not null]
  code varchar
  start_date date
  end_date date
  capacity integer
  is_active boolean [default: true]
  created_by integer [ref: > users.id]
  created_at timestamp
  updated_at timestamp
}

Table rounds {
  id integer [pk, increment]
  org_id integer [not null, ref: > organizations.id]
  school_id integer [ref: > schools.id]
  program_id integer [ref: > programs.id]
  batch_id integer [ref: > batches.id]
  name varchar [not null]
  code varchar
  start_date date
  end_date date
  capacity integer
  is_active boolean [default: true]
  display_order integer [default: 0]
  created_by integer [ref: > users.id]
  created_at timestamp
  updated_at timestamp
}

Table applicationForms {
  id integer [pk, increment]
  uuid uuid [unique]
  title varchar [not null]
  applicationFormName varchar
  organizationId integer [ref: > organizations.id]
  programId integer [not null, ref: > programs.id]
  cohortId integer [ref: > batches.id]
  roundId integer [ref: > rounds.id]
  fees float
  currency varchar [default: 'INR']
  skipPayment boolean
  isActive boolean [default: false]
  startDate date
  deadlineDate date
  formLogoUrl varchar
  formBannerUrl varchar
  formInstruction text
  formOrder integer
  enableInterviews boolean [default: false]
  showCoupon boolean [default: false]
  hideNavbar boolean [default: false]
  studentPortalLink varchar
  createdBy integer [ref: > users.id]
  updatedBy integer [ref: > users.id]
  created_at timestamp
  updated_at timestamp
}

// ============================================
// V2 LEAD MANAGEMENT (Two-Tier Architecture)
// ============================================

Table v2_leads {
  id integer [pk, increment, note: 'Shared hub for all leads']
  uuid uuid [unique]
  org_id integer [not null, ref: > organizations.id]
  school_id integer [ref: > schools.id]
  program_id integer [ref: > programs.id]
  batch_id integer [ref: > batches.id]
  round_id integer [ref: > rounds.id]
  form_id integer [ref: > applicationForms.id]
  user_id integer [ref: > users.id]
  lead_table_id integer [not null, ref: > org_lead_tables.id, note: 'FK to org_lead_tables — which org table stores org-specific data']

  // --- Contact ---
  registered_name varchar
  registered_email varchar
  registered_mobile varchar
  country_code varchar

  // --- Status & Pipeline ---
  status varchar [default: 'new']
  lead_score double
  lead_stage_id integer
  lead_sub_stage_id integer
  counsellor_id integer

  // --- Verification ---
  is_mobile_verified boolean [default: false]
  is_email_verified boolean [default: false]
  alternate_email text
  alternate_mobile_number varchar

  // --- Tracking / UTM ---
  source varchar
  medium varchar
  campaign varchar
  platform varchar
  lead_origin varchar
  lead_device varchar
  is_organic boolean
  gclid text
  fbclid text
  fb_lead_id varchar
  utm_term text
  utm_content text
  utm_campaign_id text
  utm_ad_group_id text

  // --- Payment (common across all orgs) ---
  is_payment_done boolean [default: false]
  payment_status varchar
  payment_mode varchar
  total_amount decimal
  payment_initiated boolean [default: false]
  payment_completed_at timestamp
  payment_failure_reason text
  payment_first_initiated_at timestamp
  payment_last_initiated_at timestamp
  payment_method varchar
  payment_partner varchar

  // --- Metadata ---
  created_by integer [ref: > users.id]
  created_at timestamp
  updated_at timestamp

  indexes {
    org_id [name: 'idx_v2_leads_org']
    user_id [name: 'idx_v2_leads_user']
    registered_email [name: 'idx_v2_leads_email']
    registered_mobile [name: 'idx_v2_leads_mobile']
    (org_id, program_id) [name: 'idx_v2_leads_program']
    (org_id, school_id) [name: 'idx_v2_leads_school']
    (org_id, status) [name: 'idx_v2_leads_status']
    (org_id, lead_stage_id) [name: 'idx_v2_leads_stage']
    (org_id, counsellor_id) [name: 'idx_v2_leads_counsellor']
    lead_table_id [name: 'idx_v2_leads_lead_table']
    (org_id, user_id, program_id, form_id) [unique, name: 'v2_leads_org_user_program_form_unique', note: 'Partial: WHERE user_id IS NOT NULL']
  } 
}

Table pgp_leads {
  id integer [pk, increment, note: 'Masters Union PGP org-specific fields']
  org_id integer [not null, ref: > organizations.id]
  lead_id integer [ref: > v2_leads.id, note: 'FK to shared hub']

  // --- Personal Details ---
  title varchar
  date_of_birth date
  gender varchar
  country_of_birth varchar
  linkedin_url text
  personal_website_url text
  has_foreign_citizenship boolean
  foreign_citizenship_country varchar
  foreign_citizenship_status varchar
  has_applied_before boolean
  has_physical_disability boolean
  disability_type varchar
  preferred_interview_location varchar

  // --- Current Work ---
  has_full_time_experience boolean
  current_work_type varchar
  current_family_business_name varchar
  current_function varchar
  current_industry varchar
  current_work_start_date date
  current_work_end_date date
  current_designation varchar
  current_experience_months integer
  current_role_description text

  // --- Family Business ---
  family_business_is_listed boolean
  family_business_employee_count integer
  family_business_annual_revenue decimal
  family_business_annual_salary decimal

  // --- Totals ---
  total_work_experience_months integer
  cv_url text

  // --- GMAT ---
  gmat_overall integer
  gmat_quant integer
  gmat_verbal integer
  gmat_ir integer
  gmat_awa decimal

  // --- GMAT Focus ---
  gmat_focus_overall_percentile integer
  gmat_focus_overall_score integer
  gmat_focus_quant_percentile integer
  gmat_focus_quant_score integer
  gmat_focus_verbal_percentile integer
  gmat_focus_verbal_score integer
  gmat_focus_di_percentile integer
  gmat_focus_di_score integer

  // --- CAT ---
  cat_result_status varchar
  cat_overall_percentile decimal
  cat_overall_score decimal
  cat_varc_percentile decimal
  cat_varc_score decimal
  cat_dilr_percentile decimal
  cat_dilr_score decimal
  cat_quant_percentile decimal
  cat_quant_score decimal

  // --- Other ---
  top_3_things text
  declaration_name varchar
  declaration_date date
  criminal_declaration boolean
  terms_accepted boolean

  // --- Address ---
  address text
  pin_code varchar
  country varchar
  state varchar
  city varchar

  // --- Control Fields ---
  has_additional_experience boolean
  previous_work_types varchar
  has_internship_experience boolean
  entrance_exams_taken varchar

  // --- Previous Work ×5 (9 fields each = 45) ---
  // prev_work{1-5}_type, _organization_name, _industry, _function,
  // _designation, _start_date, _end_date, _experience_months, _annual_revenue
  prev_work1_type varchar [note: 'Slots 1-5: Full-Time/Part-Time/Contract']
  prev_work1_organization_name varchar
  prev_work1_industry varchar
  prev_work1_function varchar
  prev_work1_designation varchar
  prev_work1_start_date date
  prev_work1_end_date date
  prev_work1_experience_months integer
  prev_work1_annual_revenue decimal
  // ... prev_work2_* through prev_work5_* (same pattern)

  // --- Internships ×5 (8 fields each = 40) ---
  // intern{1-5}_organization_name, _industry, _function,
  // _designation, _start_date, _end_date, _experience_months, _monthly_stipend
  intern1_organization_name varchar [note: 'Slots 1-5']
  intern1_industry varchar
  intern1_function varchar
  intern1_designation varchar
  intern1_start_date date
  intern1_end_date date
  intern1_experience_months integer
  intern1_monthly_stipend decimal
  // ... intern2_* through intern5_* (same pattern)

  // --- Certifications ×5 (6 fields each = 30) ---
  // cert{1-5}_name, _institute, _status, _rank, _level, _year_of_passing
  cert1_name varchar [note: 'Slots 1-5']
  cert1_institute varchar
  cert1_status varchar
  cert1_rank varchar
  cert1_level varchar
  cert1_year_of_passing integer
  // ... cert2_* through cert5_* (same pattern)

  // --- Awards ×5 (5 fields each = 25) ---
  // award{1-5}_name, _awarding_body, _year, _document_url, _description
  award1_name varchar [note: 'Slots 1-5']
  award1_awarding_body varchar
  award1_year integer
  award1_document_url text
  award1_description text
  // ... award2_* through award5_* (same pattern)

  // --- Documents (5) ---
  doc_aadhaar_url text
  doc_pan_url text
  doc_marksheet_10_12_url text
  doc_ug_marksheet_url text
  doc_payslips_url text

  created_at timestamp
  updated_at timestamp

  indexes {
    org_id [name: 'pgp_leads_org_id_idx']
    lead_id [name: 'pgp_leads_lead_id_idx']
  }

  Note: 'Total: ~206 columns (57 original + 149 flat). Payment fields moved to v2_leads. Repeatable sections use numbered suffixes (prev_work1-5, intern1-5, cert1-5, award1-5) instead of satellite tables.'
}

Table leads {
  id integer [pk, increment, note: 'Anandi org lead table (V1)']
  org_id integer [ref: > organizations.id]
  lead_id integer [ref: > v2_leads.id, note: 'FK to shared hub (added by V2)']
  fb_lead_id varchar

  // --- Contact ---
  registeredEmail varchar
  registeredName varchar
  registeredMobile varchar
  countryCode varchar

  // --- Child Info (Anandi-specific) ---
  child_full_name varchar
  child_age varchar
  date_of_birth date
  current_school varchar
  current_grade varchar
  admission_grade varchar

  // --- Status ---
  status varchar
  lead_stage_id integer
  counsellor_id integer [ref: > counsellors.id]
  program_id integer [ref: > programs.id]

  // --- Parent 1 (flat) ---
  parent1_name varchar
  parent1_email varchar
  parent1_phone varchar
  parent1_phone_code varchar
  parent1_relationship varchar
  parent1_occupation varchar
  parent1_qualification varchar
  parent1_annual_income varchar

  // --- Parent 2 (flat) ---
  parent2_name varchar
  parent2_email varchar
  parent2_phone varchar
  parent2_phone_code varchar
  parent2_relationship varchar
  parent2_occupation varchar
  parent2_qualification varchar
  parent2_annual_income varchar

  created_at timestamp
  updated_at timestamp
}

// ============================================
// DYNAMIC SCHEMA REGISTRY
// ============================================

Table org_lead_tables {
  id integer [pk, increment]
  org_id integer [not null, ref: > organizations.id]
  school_id integer [ref: > schools.id]
  table_name varchar [not null, note: 'Actual DB table name']
  type varchar [not null, note: 'lead or satellite']
  is_active boolean [default: true]
  created_at timestamp
  updated_at timestamp

  indexes {
    (org_id, school_id, table_name) [unique, name: 'org_lead_tables_org_school_table_unique']
  }
}

Table org_lead_columns {
  id integer [pk, increment]
  table_id integer [not null, ref: > org_lead_tables.id]
  org_id integer [not null, ref: > organizations.id]
  column_name varchar [not null, note: 'Actual DB column name (snake_case)']
  column_alias varchar [note: 'Frontend display label']
  is_required boolean [default: false]
  is_filterable boolean [default: false]
  col_order integer [note: 'Display/download order']
  is_common boolean [default: false, note: 'Lives in v2_leads, alias only']
  created_at timestamp
  updated_at timestamp

  indexes {
    (table_id, column_name) [unique, name: 'org_lead_columns_table_id_column_name_unique']
  }
}

// ============================================
// SATELLITE TABLES (linked to leads via lead_id)
// ============================================

Table parents {
  id integer [pk, increment, note: 'Legacy V1 table — Anandi now uses flat parent1_*/parent2_* columns in leads']
  lead_id integer [note: 'References leads.id (FK dropped for V2 flexibility)']
  org_id integer
  name varchar
  email varchar
  phone varchar
  relationship varchar
  occupation varchar
  created_at timestamp
  updated_at timestamp
}

Table educations {
  id integer [pk, increment]
  lead_id integer [note: 'References leads.id (FK dropped for V2 flexibility)']
  org_id integer
  display_order integer [default: 0]
  degree_name varchar
  university_name varchar
  level varchar
  board varchar
  score varchar
  created_at timestamp
  updated_at timestamp
}
