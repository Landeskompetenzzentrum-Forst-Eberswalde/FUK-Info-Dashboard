<script setup>
    import { ref, onMounted, getCurrentInstance, useAttrs } from 'vue'
    import { createClient } from '@supabase/supabase-js'

    const access_token = ref(null);
    const user_email = ref(null);
    
    const instance = getCurrentInstance();
    const apikey = instance.appContext.config.globalProperties.$apikey;
    const url = instance.appContext.config.globalProperties.$url;

    const supabase = createClient(url, apikey)

    supabase.auth.onAuthStateChange((event, session) => {
        access_token.value = session?.access_token;
        user_email.value = session?.user?.email;
    });
</script>

<template>
    <div v-if="access_token">
        <slot />
    </div>
    <div v-else>
        <p>
            You need to be logged in to see this content.
        </p>
        <p>
            <a href="/authentication/sign-in" class="VPLink">
                <v-btn rounded="xl" variant="tonal">
                    <span>Sign in</span>
                </v-btn>
            </a>
       </p>
    </div>
</template>