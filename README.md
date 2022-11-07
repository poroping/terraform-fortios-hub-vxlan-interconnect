<!-- BEGIN_TF_DOCS -->
# terraform-fortios-hub-vxlan-interconnect

Creates VXLAN tunnel overlay and BGP peering between two A-A Fortigate pairs in different Azure regions.

Requires the VNETs be peered with forwarded traffic allowed.

Uses forked version of fortios provider.

Requires FortiOS >= 6.4.5

Uses sub-table resources in BGP and SDWAN parent tables. Do not mix and match here.

Intended for use with my other modules for Azure.

### Example Usage:
```hcl
terraform {
  required_version = ">= 1.0.1"
  backend "local" {}
  required_providers {
    fortios = {
      source  = "poroping/fortios"
      version = ">= 3.1.4"
    }
  }
}

locals { 
  api_key = "keys" 
}

provider "fortios" {
  alias    = "eu_hub1"
  hostname = "20.223.1.1:10443"
  token    = local.api_key
  vdom     = "root"
  insecure = "true"
}

provider "fortios" {
  alias    = "eu_hub2"
  hostname = "20.223.1.2:10443"
  token    = local.api_key
  vdom     = "root"
  insecure = "true"
}

provider "fortios" {
  alias    = "us_hub1"
  hostname = "52.224.1.1:10443"
  token    = local.api_key
  vdom     = "root"
  insecure = "true"
}

provider "fortios" {
  alias    = "us_hub2"
  hostname = "52.224.1.2:10443"
  token    = local.api_key
  vdom     = "root"
  insecure = "true"
}

locals {
  tunnel_supernet = "10.254.0.0/24"
}

resource "fortios_system_sdwan_zone" "region1_hub1" {
  provider = fortios.eu_hub1

  allow_append = true

  name                  = "VX-EU-US"
  service_sla_tie_break = "fib-best-match"
}

resource "fortios_system_sdwan_zone" "region1_hub2" {
  provider = fortios.eu_hub2

  allow_append = true

  name                  = "VX-EU-US"
  service_sla_tie_break = "fib-best-match"
}

resource "fortios_system_sdwan_zone" "region2_hub1" {
  provider = fortios.us_hub1

  allow_append = true

  name                  = "VX-US-EU"
  service_sla_tie_break = "fib-best-match"
}

resource "fortios_system_sdwan_zone" "region2_hub2" {
  provider = fortios.us_hub2

  allow_append = true

  name                  = "VX-US-EU"
  service_sla_tie_break = "fib-best-match"
}

module "hub_interconnect-1-1" {
  providers = {
    fortios.a = fortios.eu_hub1
    fortios.b = fortios.us_hub1
  }
  source  = "poroping/hub-vxlan-interconnect/fortios"
  version = "0.0.1"

  side_a = {
    local_gateway    = null
    parent_interface = "port1"
    bgp_as           = 64420
    sdwan_zone       = fortios_system_sdwan_zone.region1_hub1.name
  }
  side_b = {
    local_gateway    = null
    parent_interface = "port1"
    bgp_as           = 64430
    sdwan_zone       = fortios_system_sdwan_zone.region2_hub1.name
  }
  tunnel_subnet  = cidrsubnet(local.tunnel_supernet, 7, 0)
  name_overwrite = "VX-1"
}

module "hub_interconnect-1-2" {
  providers = {
    fortios.a = fortios.eu_hub1
    fortios.b = fortios.us_hub2
  }
  source  = "poroping/hub-vxlan-interconnect/fortios"
  version = "0.0.1"

  side_a = {
    local_gateway    = null
    parent_interface = "port1"
    bgp_as           = 64420
    sdwan_zone       = fortios_system_sdwan_zone.region1_hub1.name
  }
  side_b = {
    local_gateway    = null
    parent_interface = "port1"
    bgp_as           = 64430
    sdwan_zone       = fortios_system_sdwan_zone.region2_hub2.name
  }
  tunnel_subnet  = cidrsubnet(local.tunnel_supernet, 7, 1)
  name_overwrite = "VX-2"
}

module "hub_interconnect-2-1" {
  providers = {
    fortios.a = fortios.eu_hub2
    fortios.b = fortios.us_hub1
  }
  source  = "poroping/hub-vxlan-interconnect/fortios"
  version = "0.0.1"

  side_a = {
    local_gateway    = null
    parent_interface = "port1"
    bgp_as           = 64420
    sdwan_zone       = fortios_system_sdwan_zone.region1_hub2.name
  }
  side_b = {
    local_gateway    = null
    parent_interface = "port1"
    bgp_as           = 64430
    sdwan_zone       = fortios_system_sdwan_zone.region2_hub1.name
  }
  tunnel_subnet  = cidrsubnet(local.tunnel_supernet, 7, 2)
  name_overwrite = "VX-2"
}

module "hub_interconnect-2-2" {
  providers = {
    fortios.a = fortios.eu_hub2
    fortios.b = fortios.us_hub2
  }
  source  = "poroping/hub-vxlan-interconnect/fortios"
  version = "0.0.1"

  side_a = {
    local_gateway    = null
    parent_interface = "port1"
    bgp_as           = 64420
    sdwan_zone       = fortios_system_sdwan_zone.region1_hub2.name
  }
  side_b = {
    local_gateway    = null
    parent_interface = "port1"
    bgp_as           = 64430
    sdwan_zone       = fortios_system_sdwan_zone.region2_hub2.name
  }
  tunnel_subnet  = cidrsubnet(local.tunnel_supernet, 7, 3)
  name_overwrite = "VX-1"
}
```

## Providers

| Name | Version |
|------|---------|
| <a name="provider_fortios.a"></a> [fortios.a](#provider\_fortios.a) | >= 3.1.5 |
| <a name="provider_fortios.b"></a> [fortios.b](#provider\_fortios.b) | >= 3.1.5 |
| <a name="provider_random"></a> [random](#provider\_random) | n/a |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_side_a"></a> [side\_a](#input\_side\_a) | n/a | <pre>object({<br>    local_gateway    = string<br>    parent_interface = string<br>    bgp_as           = number<br>    sdwan_zone       = string<br>  })</pre> | n/a | yes |
| <a name="input_side_b"></a> [side\_b](#input\_side\_b) | n/a | <pre>object({<br>    local_gateway    = string<br>    parent_interface = string<br>    bgp_as           = number<br>    sdwan_zone       = string<br>  })</pre> | n/a | yes |
| <a name="input_name_overwrite"></a> [name\_overwrite](#input\_name\_overwrite) | n/a | `string` | `null` | no |
| <a name="input_tunnel_subnet"></a> [tunnel\_subnet](#input\_tunnel\_subnet) | n/a | `string` | `"10.254.0.0/31"` | no |


<!-- END_TF_DOCS -->    