gcloud compute routes create custom_route_name \
   --network=name_of_spoke_vpc \
   --destination-range=0.0.0.0/0 \
   --next-hop-ilb=ip_address_of_ilb \
   --priority=custom_route_priority \
   --tags=network_tag_value