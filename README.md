# CoreDNS Request for Comments

The purpose of CoreDNS RFC is to help community involvement in discussions
and feedbacks in case of a significant addition or change in CoreDNS and
affiliated repos in CoreDNS org. 

## Who is involved?

Any **community member** may help by providing feedback on whether the RFC will
meet their needs.

An **RFC author** is one or more community member who writes an RFC and is
committed to championing it through the  process.

An **RFC sponsor** is any CoreDNS maintainer who sponsors the RFC and will shepherd it
through the RFC review process.

## RFC process

1. Recruit a sponsor from CoreDNS maintainers.

   You can open an issue in this repo before creating the pull
   request for RFC.

2. Submit your RFC as a pull request. 

   Name your RFC file using the [template](https://github.com/coredns/rfc/blob/master/yyyymmdd-rfc-template.md) `YYYYMMDD-descriptive-name.md`, where
   YYYYMMDD is the date of submission, and ‘descriptive-name’ relates to the
   title of your RFC. For instance, if your RFC is titled “A Special Plugin for CoreDNS”,
   you might use the filename `20180531-special-plugin.md`. If you have images
   or other auxiliary files, create a directory of the form `YYYYMMDD-descriptive-name`
   in which to store those files.

3. Ping related maintainers and encourage community discussions and feedbacks.
   PR and a request for review. Follow the example of previous mailings,
   as you can see in [this
   example](https://groups.google.com/a/tensorflow.org/forum/#!topic/developers/PIChGLLnpTE).

4. The pull request should be kept open for at least two weeks to allow feedback from
   community. All questions raised by community should be settled, and participating
   CoreDNS maintainers should reach a consensus before the RFC is merged.

5. A follow up implementation is expected to happen once RFC is merged. Otherwise,
   the status of the RFC could be changed to `Obsolete` if not actively worked on.

## RFC Template

Use the template [from
GitHub](https://github.com/coredns/rfc/blob/master/yyyymmdd-rfc-template.md),
being sure to follow the naming conventions described above.
