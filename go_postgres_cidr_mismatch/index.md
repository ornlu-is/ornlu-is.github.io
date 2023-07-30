# CIDRs and how they are handled by different systems


I came across an interesting bug in the past few days. I had a very simple Go program that had a single purpose: it would take some user input, process that input, and then write it to a database. However, the program would sometimes fail, when given input that apparently was valid. And I thought that this bug was interesting enough to write about it, so here we are.

## Understanding CIDR notation

Before diving into the actual bug, it is fundamental that we understand CIDR notation. Classless Internet Domain Routing, or CIDR, is notation used to define subnetworks, which is composed of an IP address followed by a forward slash and a number between zero and thirty two, *e.g.*, 6.6.6.0/24. From a given CIDR, we can extract some information about the subnetwork:
* Network ID - this is what allows us to identify the network and corresponds to the IP address represented in the CIDR. In the example CIDR given above, this would be 6.6.6.0;
* Broadcast IP - this is the IP address that is used to broadcast messages to IP addresses in the network and corresponds to the last IP address in the IP range. for our example, it would be 6.6.6.255;
* First useable host IP - obviously, the actual first IP is the network ID, but that IP is not useable. However, the IP immediately after that one can actually be used and, in the context of our example, it would 6.6.6.1; 
* Last useable host IP - similartly to the first useable host IP, the actual last IP is not useable as a host, but the one immediately before it is! So, for our running example, the last useable host IP would be 6.6.6.255;
* Number of IP addresses - this is fairly easy to calculate from the CIDR notation. Simply take the number that follows the forward slash, subtract it to 32, then take the power of 2 of the corresponding result. *E.g.*, $32 - 24 = 8$, $2^8 = 256$, so for a /24 subnetwork, we have 256 IP addresses. 

## The bug

The functioning of my program was very simple. Programmed in Go, it would take a user given CIDR, parse it, and then write that CIDR into a Postgres database. Most of the times, this program would work perfectly, as expected. But, sometimes, it would fail. For example, for the input "6.6.6.0/24", the program would function normally, but, for the input "3.3.1.0/16", Postgres would throw an error.

Now, there are two possible places where this error might be coming from: either from the Go application or the Postgres database. The Postgres table where the data was being written had a column of type `cidr`. From inspecting the documentation on Postgres network address types, we see that we really only had two options for this: `cidr` or `inet`, with the main difference between the two of them being that `cidr` does not allow for non-zero bits to the right of the network mask. 

With this bit of information, it should already be obvious what was going on: "3.3.1.0/16" is not a proper CIDR! The correct CIDR for the network it is trying to represent would be "3.3.0.0/16". 

## Golang CIDR parsing

It is now very clear why Postgres was throwing an error, but why did this get through our Go application? The code snippet responsible for checking the CIDRs looks something like this:

```go
func checkCIDR(givenCIDR string) (*net.IPNet, error) {
	addr, inet, err := net.ParseCIDR(givenCIDR)
	if err != nil {
		return nil, fmt.Errorf("error parsing CIDR: %w", err)
	}

	return &net.IPNet{
		IP:   addr,
		Mask: inet.Mask,
	}, nil
}
```

If you take this code snippet and try to run it with the improper CIDR "3.3.1.0./16", you'll see that the code runs without returning an error! This is because the `ParseCIDR` function of the `net` package infers the subnet the CIDR refers to even if it is not a proper CIDR. If you look carefully at its returned values, we get both an IP address and an IP subnet. The IP address will correspond to the IP address given in the, sometimes improper, CIDR while the IP network will correspond to the inferred network. Meaning that Go would calculate the inferred network, but when we returned the value, we would return the improper CIDR, which then led to a Postgres error.

## The fix

There were two possible ways of fixing this: either allowing users to specify improper CIDRs and having the application infer the correct network, or have the application reject all improper CIDRs. Ultimately, this is more of a user experience question. I prefer forcing users to know what they are playing around with, so, in my opinion, the fix should be to reject all improper CIDRs. This means that, we need to change the function that checks the CIDR to the following:

```go
func checkCIDR(givenCIDR string) (*net.IPNet, error) {
	addr, inet, err := net.ParseCIDR(givenCIDR)
	if err != nil {
		return nil, fmt.Errorf("error parsing CIDR: %w", err)
	}

	if !addr.Equal(inet.IP) {
		return nil, fmt.Errorf("invalid CIDR provided")
	}

	return &net.IPNet{
		IP:   addr,
		Mask: inet.Mask,
	}, nil
}
```

You might, obviously, not want to take the same approach to fix this as I did, and that is completely fine: there is no universally correct approach to tackle this bug. Nonetheless, I thought this bug was a great learning opportunity and hopefully so do you.

## References

* Postgres documentation on network address types - https://www.postgresql.org/docs/current/datatype-net-types.html
* Golang `net` package documentation - https://pkg.go.dev/net

