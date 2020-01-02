9.  Connection Migration

   The use of a connection ID allows connections to survive changes to
   endpoint addresses (IP address and port), such as those caused by an
   endpoint migrating to a new network.  This section describes the
   process by which an endpoint migrates to a new address.

   The design of QUIC relies on endpoints retaining a stable address for
   the duration of the handshake.  An endpoint MUST NOT initiate
   connection migration before the handshake is confirmed, as defined in
   section 4.1.2 of [QUIC-TLS].
An endpoint also MUST NOT send packets from a different local
   address, actively initiating migration, if the peer sent the
   "disable_active_migration" transport parameter during the handshake.
   An endpoint which has sent this transport parameter, but detects that
   a peer has nonetheless migrated to a different network MUST either
   drop the incoming packets on that path without generating a stateless
   reset or proceed with path validation and allow the peer to migrate.
   Generating a stateless reset or closing the connection would allow
   third parties in the network to cause connections to close by
   spoofing or otherwise manipulating observed traffic.

   Not all changes of peer address are intentional, or active,
   migrations.  The peer could experience NAT rebinding: a change of
   address due to a middlebox, usually a NAT, allocating a new outgoing
   port or even a new outgoing IP address for a flow.  An endpoint MUST
   perform path validation (Section 8.2) if it detects any change to a
   peer's address, unless it has previously validated that address.

   When an endpoint has no validated path on which to send packets, it
   MAY discard connection state.  An endpoint capable of connection
   migration MAY wait for a new path to become available before
   discarding connection state.

   This document limits migration of connections to new client
   addresses, except as described in Section 9.6.  Clients are
   responsible for initiating all migrations.  Servers do not send non-
   probing packets (see Section 9.1) toward a client address until they
   see a non-probing packet from that address.  If a client receives
   packets from an unknown server address, the client MUST discard these
   packets.

9.1.  Probing a New Path

   An endpoint MAY probe for peer reachability from a new local address
   using path validation Section 8.2 prior to migrating the connection
   to the new local address.  Failure of path validation simply means
   that the new path is not usable for this connection.  Failure to
   validate a path does not cause the connection to end unless there are
   no valid alternative paths available.

   An endpoint uses a new connection ID for probes sent from a new local
   address, see Section 9.5 for further discussion.  An endpoint that
   uses a new local address needs to ensure that at least one new
   connection ID is available at the peer.  That can be achieved by
   including a NEW_CONNECTION_ID frame in the probe.
 Receiving a PATH_CHALLENGE frame from a peer indicates that the peer
   is probing for reachability on a path.  An endpoint sends a
   PATH_RESPONSE in response as per Section 8.2.

   PATH_CHALLENGE, PATH_RESPONSE, NEW_CONNECTION_ID, and PADDING frames
   are "probing frames", and all other frames are "non-probing frames".
   A packet containing only probing frames is a "probing packet", and a
   packet containing any other frame is a "non-probing packet".

9.2.  Initiating Connection Migration

   An endpoint can migrate a connection to a new local address by
   sending packets containing non-probing frames from that address.

   Each endpoint validates its peer's address during connection
   establishment.  Therefore, a migrating endpoint can send to its peer
   knowing that the peer is willing to receive at the peer's current
   address.  Thus an endpoint can migrate to a new local address without
   first validating the peer's address.

   When migrating, the new path might not support the endpoint's current
   sending rate.  Therefore, the endpoint resets its congestion
   controller, as described in Section 9.4.

   The new path might not have the same ECN capability.  Therefore, the
   endpoint verifies ECN capability as described in Section 13.4.

   Receiving acknowledgments for data sent on the new path serves as
   proof of the peer's reachability from the new address.  Note that
   since acknowledgments may be received on any path, return
   reachability on the new path is not established.  To establish return
   reachability on the new path, an endpoint MAY concurrently initiate
   path validation Section 8.2 on the new path.

9.3.  Responding to Connection Migration

   Receiving a packet from a new peer address containing a non-probing
   frame indicates that the peer has migrated to that address.

   In response to such a packet, an endpoint MUST start sending
   subsequent packets to the new peer address and MUST initiate path
   validation (Section 8.2) to verify the peer's ownership of the
   unvalidated address.

   An endpoint MAY send data to an unvalidated peer address, but it MUST
   protect against potential attacks as described in Section 9.3.1 and
   Section 9.3.2.  An endpoint MAY skip validation of a peer address if
   that address has been seen recently.


An endpoint only changes the address that it sends packets to in
   response to the highest-numbered non-probing packet.  This ensures
   that an endpoint does not send packets to an old peer address in the
   case that it receives reordered packets.

   After changing the address to which it sends non-probing packets, an
   endpoint could abandon any path validation for other addresses.

   Receiving a packet from a new peer address might be the result of a
   NAT rebinding at the peer.

   After verifying a new client address, the server SHOULD send new
   address validation tokens (Section 8) to the client.

9.3.1.  Peer Address Spoofing

   It is possible that a peer is spoofing its source address to cause an
   endpoint to send excessive amounts of data to an unwilling host.  If
   the endpoint sends significantly more data than the spoofing peer,
   connection migration might be used to amplify the volume of data that
   an attacker can generate toward a victim.

   As described in Section 9.3, an endpoint is required to validate a
   peer's new address to confirm the peer's possession of the new
   address.  Until a peer's address is deemed valid, an endpoint MUST
   limit the rate at which it sends data to this address.  The endpoint
   MUST NOT send more than a minimum congestion window's worth of data
   per estimated round-trip time (kMinimumWindow, as defined in
   [QUIC-RECOVERY]).  In the absence of this limit, an endpoint risks
   being used for a denial of service attack against an unsuspecting
   victim.  Note that since the endpoint will not have any round-trip
   time measurements to this address, the estimate SHOULD be the default
   initial value (see [QUIC-RECOVERY]).

   If an endpoint skips validation of a peer address as described in
   Section 9.3, it does not need to limit its sending rate.

9.3.2.  On-Path Address Spoofing

   An on-path attacker could cause a spurious connection migration by
   copying and forwarding a packet with a spoofed address such that it
   arrives before the original packet.  The packet with the spoofed
   address will be seen to come from a migrating connection, and the
   original packet will be seen as a duplicate and dropped.  After a
   spurious migration, validation of the source address will fail
   because the entity at the source address does not have the necessary
   cryptographic keys to read or respond to the PATH_CHALLENGE frame
   that is sent to it even if it wanted to.

   To protect the connection from failing due to such a spurious
   migration, an endpoint MUST revert to using the last validated peer
   address when validation of a new peer address fails.

   If an endpoint has no state about the last validated peer address, it
   MUST close the connection silently by discarding all connection
   state.  This results in new packets on the connection being handled
   generically.  For instance, an endpoint MAY send a stateless reset in
   response to any further incoming packets.

   Note that receipt of packets with higher packet numbers from the
   legitimate peer address will trigger another connection migration.
   This will cause the validation of the address of the spurious
   migration to be abandoned.

9.3.3.  Off-Path Packet Forwarding

   An off-path attacker that can observe packets might forward copies of
   genuine packets to endpoints.  If the copied packet arrives before
   the genuine packet, this will appear as a NAT rebinding.  Any genuine
   packet will be discarded as a duplicate.  If the attacker is able to
   continue forwarding packets, it might be able to cause migration to a
   path via the attacker.  This places the attacker on path, giving it
   the ability to observe or drop all subsequent packets.

   Unlike the attack described in Section 9.3.2, the attacker can ensure
   that the new path is successfully validated.

   This style of attack relies on the attacker using a path that is
   approximately as fast as the direct path between endpoints.  The
   attack is more reliable if relatively few packets are sent or if
   packet loss coincides with the attempted attack.

   A non-probing packet received on the original path that increases the
   maximum received packet number will cause the endpoint to move back
   to that path.  Eliciting packets on this path increases the
   likelihood that the attack is unsuccessful.  Therefore, mitigation of
   this attack relies on triggering the exchange of packets.

   In response to an apparent migration, endpoints MUST validate the
   previously active path using a PATH_CHALLENGE frame.  This induces
   the sending of new packets on that path.  If the path is no longer
   viable, the validation attempt will time out and fail; if the path is
   viable, but no longer desired, the validation will succeed, but only
   results in probing packets being sent on the path.

   An endpoint that receives a PATH_CHALLENGE on an active path SHOULD
   send a non-probing packet in response.  If the non-probing packet
arrives before any copy made by an attacker, this results in the
   connection being migrated back to the original path.  Any subsequent
   migration to another path restarts this entire process.

   This defense is imperfect, but this is not considered a serious
   problem.  If the path via the attack is reliably faster than the
   original path despite multiple attempts to use that original path, it
   is not possible to distinguish between attack and an improvement in
   routing.

   An endpoint could also use heuristics to improve detection of this
   style of attack.  For instance, NAT rebinding is improbable if
   packets were recently received on the old path, similarly rebinding
   is rare on IPv6 paths.  Endpoints can also look for duplicated
   packets.  Conversely, a change in connection ID is more likely to
   indicate an intentional migration rather than an attack.

9.4.  Loss Detection and Congestion Control

   The capacity available on the new path might not be the same as the
   old path.  Packets sent on the old path SHOULD NOT contribute to
   congestion control or RTT estimation for the new path.

   On confirming a peer's ownership of its new address, an endpoint MUST
   immediately reset the congestion controller and round-trip time
   estimator for the new path to initial values (see Sections A.3 and
   B.3 in [QUIC-RECOVERY]) unless it has knowledge that a previous send
   rate or round-trip time estimate is valid for the new path.  For
   instance, an endpoint might infer that a change in only the client's
   port number is indicative of a NAT rebinding, meaning that the new
   path is likely to have similar bandwidth and round-trip time.
   However, this determination will be imperfect.  If the determination
   is incorrect, the congestion controller and the RTT estimator are
   expected to adapt to the new path.  Generally, implementations are
   advised to be cautious when using previous values on a new path.

   There may be apparent reordering at the receiver when an endpoint
   sends data and probes from/to multiple addresses during the migration
   period, since the two resulting paths may have different round-trip
   times.  A receiver of packets on multiple paths will still send ACK
   frames covering all received packets.

   While multiple paths might be used during connection migration, a
   single congestion control context and a single loss recovery context
   (as described in [QUIC-RECOVERY]) may be adequate.  For instance, an
   endpoint might delay switching to a new congestion control context
   until it is confirmed that an old path is no longer needed (such as
   the case in Section 9.3.3).

A sender can make exceptions for probe packets so that their loss
   detection is independent and does not unduly cause the congestion
   controller to reduce its sending rate.  An endpoint might set a
   separate timer when a PATH_CHALLENGE is sent, which is cancelled when
   the corresponding PATH_RESPONSE is received.  If the timer fires
   before the PATH_RESPONSE is received, the endpoint might send a new
   PATH_CHALLENGE, and restart the timer for a longer period of time.

9.5.  Privacy Implications of Connection Migration

   Using a stable connection ID on multiple network paths allows a
   passive observer to correlate activity between those paths.  An
   endpoint that moves between networks might not wish to have their
   activity correlated by any entity other than their peer, so different
   connection IDs are used when sending from different local addresses,
   as discussed in Section 5.1.  For this to be effective endpoints need
   to ensure that connection IDs they provide cannot be linked by any
   other entity.

   At any time, endpoints MAY change the Destination Connection ID they
   send to a value that has not been used on another path.

   An endpoint MUST use a new connection ID if it initiates connection
   migration as described in Section 9.2 or probes a new network path as
   described in Section 9.1.  An endpoint MUST use a new connection ID
   in response to a change in the address of a peer if the packet with
   the new peer address uses an active connection ID that has not been
   previously used by the peer.

   Using different connection IDs for packets sent in both directions on
   each new network path eliminates the use of the connection ID for
   linking packets from the same connection across different network
   paths.  Header protection ensures that packet numbers cannot be used
   to correlate activity.  This does not prevent other properties of
   packets, such as timing and size, from being used to correlate
   activity.

   Unintentional changes in path without a change in connection ID are
   possible.  For example, after a period of network inactivity, NAT
   rebinding might cause packets to be sent on a new path when the
   client resumes sending.

   A client might wish to reduce linkability by employing a new
   connection ID and source UDP port when sending traffic after a period
   of inactivity.  Changing the UDP port from which it sends packets at
   the same time might cause the packet to appear as a connection
   migration.  This ensures that the mechanisms that support migration
   are exercised even for clients that don't experience NAT rebindings
or genuine migrations.  Changing port number can cause a peer to
   reset its congestion state (see Section 9.4), so the port SHOULD only
   be changed infrequently.

   An endpoint that exhausts available connection IDs cannot probe new
   paths or initiate migration, nor can it respond to probes or attempts
   by its peer to migrate.  To ensure that migration is possible and
   packets sent on different paths cannot be correlated, endpoints
   SHOULD provide new connection IDs before peers migrate; see
   Section 5.1.1.  If a peer might have exhausted available connection
   IDs, a migrating endpoint could include a NEW_CONNECTION_ID frame in
   all packets sent on a new network path.

9.6.  Server's Preferred Address

   QUIC allows servers to accept connections on one IP address and
   attempt to transfer these connections to a more preferred address
   shortly after the handshake.  This is particularly useful when
   clients initially connect to an address shared by multiple servers
   but would prefer to use a unicast address to ensure connection
   stability.  This section describes the protocol for migrating a
   connection to a preferred server address.

   Migrating a connection to a new server address mid-connection is left
   for future work.  If a client receives packets from a new server
   address not indicated by the preferred_address transport parameter,
   the client SHOULD discard these packets.

9.6.1.  Communicating a Preferred Address

   A server conveys a preferred address by including the
   preferred_address transport parameter in the TLS handshake.

   Servers MAY communicate a preferred address of each address family
   (IPv4 and IPv6) to allow clients to pick the one most suited to their
   network attachment.

   Once the handshake is finished, the client SHOULD select one of the
   two server's preferred addresses and initiate path validation (see
   Section 8.2) of that address using the connection ID provided in the
   preferred_address transport parameter.

   If path validation succeeds, the client SHOULD immediately begin
   sending all future packets to the new server address using the new
   connection ID and discontinue use of the old server address.  If path
   validation fails, the client MUST continue sending all future packets
   to the server's original IP address.
9.6.2.  Responding to Connection Migration

   A server might receive a packet addressed to its preferred IP address
   at any time after it accepts a connection.  If this packet contains a
   PATH_CHALLENGE frame, the server sends a PATH_RESPONSE frame as per
   Section 8.2.  The server MUST send other non-probing frames from its
   original address until it receives a non-probing packet from the
   client at its preferred address and until the server has validated
   the new path.

   The server MUST probe on the path toward the client from its
   preferred address.  This helps to guard against spurious migration
   initiated by an attacker.

   Once the server has completed its path validation and has received a
   non-probing packet with a new largest packet number on its preferred
   address, the server begins sending non-probing packets to the client
   exclusively from its preferred IP address.  It SHOULD drop packets
   for this connection received on the old IP address, but MAY continue
   to process delayed packets.

9.6.3.  Interaction of Client Migration and Preferred Address

   A client might need to perform a connection migration before it has
   migrated to the server's preferred address.  In this case, the client
   SHOULD perform path validation to both the original and preferred
   server address from the client's new address concurrently.

   If path validation of the server's preferred address succeeds, the
   client MUST abandon validation of the original address and migrate to
   using the server's preferred address.  If path validation of the
   server's preferred address fails but validation of the server's
   original address succeeds, the client MAY migrate to its new address
   and continue sending to the server's original address.

   If the connection to the server's preferred address is not from the
   same client address, the server MUST protect against potential
   attacks as described in Section 9.3.1 and Section 9.3.2.  In addition
   to intentional simultaneous migration, this might also occur because
   the client's access network used a different NAT binding for the
   server's preferred address.

   Servers SHOULD initiate path validation to the client's new address
   upon receiving a probe packet from a different address.  Servers MUST
   NOT send more than a minimum congestion window's worth of non-probing
   packets to the new address before path validation is complete.
 A client that migrates to a new address SHOULD use a preferred
   address from the same address family for the server.

9.7.  Use of IPv6 Flow-Label and Migration

   Endpoints that send data using IPv6 SHOULD apply an IPv6 flow label
   in compliance with [RFC6437], unless the local API does not allow
   setting IPv6 flow labels.

   The IPv6 flow label SHOULD be a pseudo-random function of the source
   and destination addresses, source and destination UDP ports, and the
   destination CID.  The flow label generation MUST be designed to
   minimize the chances of linkability with a previously used flow
   label, as this would enable correlating activity on multiple paths
   (see Section 9.5).

   A possible implementation is to compute the flow label as a
   cryptographic hash function of the source and destination addresses,
   source and destination UDP ports, destination CID, and a local
   secret.


