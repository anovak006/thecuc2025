FROM freeradius/freeradius-server:3.2.8-alpine

RUN rm /etc/raddb/sites-enabled/default /etc/raddb/sites-enabled/inner-tunnel

COPY ./raddb/mods-available/eap-aai.template /opt/etc/raddb/mods-available/eap-aai
# COPY ./raddb/mods-available/ldap-aai.template /opt/etc/raddb/mods-available/ldap-aai
COPY ./raddb/mods-available/cui_log-aai /opt/etc/raddb/mods-available/cui_log-aai
COPY ./raddb/mods-available/f-ticks-aai /opt/etc/raddb/mods-available/f-ticks-aai
COPY ./raddb/policy.d/aai /opt/etc/raddb/policy.d/aai
COPY ./raddb/sites-available/aai /opt/etc/raddb/sites-available/
COPY ./raddb/clients-aai.conf /opt/etc/raddb/clients-aai.conf
COPY ./raddb/clients-local.conf /opt/etc/raddb/clients-local.conf
COPY ./raddb/proxy-aai.conf /opt/etc/raddb/proxy-aai.conf
COPY ./raddb/proxy-eduroam.conf /opt/etc/raddb/proxy-eduroam.conf

ARG raddbdir="/etc/raddb"

# Uključi module
RUN for mod in attr_filter \
        cui_log-aai \
        detail \
        detail.log \
        eap-aai \
        expr \
        f-ticks-aai \
        ldap-aai \
        preprocess \
        realm; \
    do \
        if [ ! -L ${raddbdir}/mods-enabled/${mod} ]; then \
            ln -sf ${raddbdir}/mods-available/${mod} ${raddbdir}/mods-enabled/${mod}; \
        fi; \
    done

# Isključi module
RUN for mod in always \
        cache_eap \
        chap \
        digest \
        dynamic_clients \
        eap \
        echo \
        exec \
        expiration \
        files \
        linelog \
        logintime \
        mschap \
        ntlm_auth \
        pap \
        passwd \
        radutmp \
        replicate \
        soh \
        sradutmp \
        unix \
        unpack \
        utf8; \
    do \
        if [ -L ${raddbdir}/mods-enabled/${mod} ]; then \
            rm ${raddbdir}/mods-enabled/${mod}; \
        fi; \
    done

RUN if [ ! -L ${raddbdir}/sites-enabled/aai ]; then \
      ln -sf ${raddbdir}/sites-available/aai ${raddbdir}/sites-enabled/aai; \
    fi
